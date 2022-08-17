## Return to Library
---

```c
// Name: rtl.c
// Compile: gcc -o rtl rtl.c -fno-PIE -no-pie

#include <stdio.h>
#include <unistd.h>

const char* binsh = "/bin/sh";

int main() {
  char buf[0x30];

  setvbuf(stdin, 0, _IONBF, 0);
  setvbuf(stdout, 0, _IONBF, 0);

  // Add system function to plt's entry
  system("echo 'system@plt");

  // Leak canary
  printf("[1] Leak Canary\n");
  printf("Buf: ");
  read(0, buf, 0x100);
  printf("Buf: %s\n", buf);

  // Overwrite return address
  printf("[2] Overwrite return address\n");
  printf("Buf: ");
  read(0, buf, 0x100);

  return 0;
}
```

<br><br>

## Solution
---

gdb로 확인해보니 read에서 ```rbp-0x40```부터 입력을 받으므로 입력을 ```(0x40-0x8 = 0x38)```만큼 해주면 카나리릭 코드에서 카나리를 볼 수 있다.

<br>

```shell
   0x0000000000400772 <+123>:   lea    rax,[rbp-0x40]
   0x0000000000400776 <+127>:   mov    edx,0x100
   0x000000000040077b <+132>:   mov    rsi,rax
   0x000000000040077e <+135>:   mov    edi,0x0
   0x0000000000400783 <+140>:   call   0x4005f0 <read@plt>
```

<br>

이제 ovewrite를 해줘야하는데, checksec을 통해 확인해보면 NX가 걸려있고 확인해보면 스택에 실행권한이 없음을 알 수 있다.

<br>

```shell
pwndbg> checksec
[*] '/home/index/rtl/rtl'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

<br>

걸려있는 보호기법은 ASLR, NX가 있다고 생각하면 된다.

<br>

```shell
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
          0x400000           0x401000 r-xp     1000 0      /home/index/rtl/rtl
          0x600000           0x601000 r--p     1000 0      /home/index/rtl/rtl
          0x601000           0x602000 rw-p     1000 1000   /home/index/rtl/rtl
    0x7ffff7dcb000     0x7ffff7ded000 r--p    22000 0      /usr/lib/x86_64-linux-gnu/libc-2.31.so
    0x7ffff7ded000     0x7ffff7f65000 r-xp   178000 22000  /usr/lib/x86_64-linux-gnu/libc-2.31.so
    0x7ffff7f65000     0x7ffff7fb3000 r--p    4e000 19a000 /usr/lib/x86_64-linux-gnu/libc-2.31.so
    0x7ffff7fb3000     0x7ffff7fb7000 r--p     4000 1e7000 /usr/lib/x86_64-linux-gnu/libc-2.31.so
    0x7ffff7fb7000     0x7ffff7fb9000 rw-p     2000 1eb000 /usr/lib/x86_64-linux-gnu/libc-2.31.so
    0x7ffff7fb9000     0x7ffff7fbf000 rw-p     6000 0      [anon_7ffff7fb9]
    0x7ffff7fca000     0x7ffff7fce000 r--p     4000 0      [vvar]
    0x7ffff7fce000     0x7ffff7fcf000 r-xp     1000 0      [vdso]
    0x7ffff7fcf000     0x7ffff7fd0000 r--p     1000 0      /usr/lib/x86_64-linux-gnu/ld-2.31.so
    0x7ffff7fd0000     0x7ffff7ff3000 r-xp    23000 1000   /usr/lib/x86_64-linux-gnu/ld-2.31.so
    0x7ffff7ff3000     0x7ffff7ffb000 r--p     8000 24000  /usr/lib/x86_64-linux-gnu/ld-2.31.so
    0x7ffff7ffc000     0x7ffff7ffd000 r--p     1000 2c000  /usr/lib/x86_64-linux-gnu/ld-2.31.so
    0x7ffff7ffd000     0x7ffff7ffe000 rw-p     1000 2d000  /usr/lib/x86_64-linux-gnu/ld-2.31.so
    0x7ffff7ffe000     0x7ffff7fff000 rw-p     1000 0      [anon_7ffff7ffe]
    0x7ffffffde000     0x7ffffffff000 rw-p    21000 0      [stack]
```

<br>

이 부분부터는 강의 내용을 토대로 정리..

ASLR이 걸려있으나, PIE는 걸려있지 않으므로 plt의 주소는 고정이 된다.


