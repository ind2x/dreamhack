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
pwndbg> shell cat /proc/3384/maps
00400000-00401000 r-xp 00000000 08:10 844                                /home/index/rtl/rtl
00600000-00601000 r--p 00000000 08:10 844                                /home/index/rtl/rtl
00601000-00602000 rw-p 00001000 08:10 844                                /home/index/rtl/rtl
7ffff7dcb000-7ffff7ded000 r--p 00000000 08:10 27841                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7ffff7ded000-7ffff7f65000 r-xp 00022000 08:10 27841                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7ffff7f65000-7ffff7fb3000 r--p 0019a000 08:10 27841                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7ffff7fb3000-7ffff7fb7000 r--p 001e7000 08:10 27841                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7ffff7fb7000-7ffff7fb9000 rw-p 001eb000 08:10 27841                      /usr/lib/x86_64-linux-gnu/libc-2.31.so
7ffff7fb9000-7ffff7fbf000 rw-p 00000000 00:00 0
7ffff7fca000-7ffff7fce000 r--p 00000000 00:00 0                          [vvar]
7ffff7fce000-7ffff7fcf000 r-xp 00000000 00:00 0                          [vdso]
7ffff7fcf000-7ffff7fd0000 r--p 00000000 08:10 27836                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7fd0000-7ffff7ff3000 r-xp 00001000 08:10 27836                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7ff3000-7ffff7ffb000 r--p 00024000 08:10 27836                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7ffc000-7ffff7ffd000 r--p 0002c000 08:10 27836                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7ffd000-7ffff7ffe000 rw-p 0002d000 08:10 27836                      /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7ffe000-7ffff7fff000 rw-p 00000000 00:00 0
7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0                          [stack]
```

<br>

