## rop
---

```c
// Name: rop.c
// Compile: gcc -o rop rop.c -fno-PIE -no-pie

#include <stdio.h>
#include <unistd.h>

int main() {
  char buf[0x30];

  setvbuf(stdin, 0, _IONBF, 0);
  setvbuf(stdout, 0, _IONBF, 0);

  // Leak canary
  puts("[1] Leak Canary");
  printf("Buf: ");
  read(0, buf, 0x100);
  printf("Buf: %s\n", buf);

  // Do ROP
  puts("[2] Input ROP payload");
  printf("Buf: ");
  read(0, buf, 0x100);

  return 0;
}
```

<br><br>

## Solution
---

```
[*] '/home/index/rop/rop'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

<br>

ASLR, NX가 걸려있다.

구해야 되는 값들은 다음과 같다.

+ pop rdi; ret gadget
+ puts 함수의 got 값
+ puts 함수의 오프셋
+ put@got를 이용해서 libc의 주소 == puts 함수 - puts 함수 오프셋
+ system 함수의 오프셋 -> puts@got에 overwrite 할 것임
+ ```/bin/sh``` 문자열의 주소


먼저 카나리 값을 구한 다음, puts 함수를 호출하여 puts 함수의 got 값을 알아낼 것이다.

공격은 리턴을 puts@plt로 하여 puts@got의 값을 읽어내서 이 값을 통해 libc의 
