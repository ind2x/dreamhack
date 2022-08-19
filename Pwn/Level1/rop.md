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
+ puts 함수의 got 주소 == 0x601018
+ read 함수의 오프셋
+ put@got를 이용해서 libc의 주소 == puts 함수 - puts 함수 오프셋
+ system 함수의 오프셋 -> puts@got에 overwrite 할 것임
+ ```/bin/sh``` 문자열의 주소


먼저 카나리 값을 구한 다음, puts 함수를 호출하여 puts 함수의 got 값을 알아낼 것이다.

위 문제는 PIE가 적용되어 있지 않아서 plt와 got의 주소는 동일하다.

공격은 리턴을 puts@plt로 하여 read@got의 값을 읽어내서 read 함수의 주소를 구한 뒤, 이 값을 통해 libc의 베이스 주소를 알아내서 system 함수의 주소를 알아내고, 이번엔 read 함수를 호출하여 read@got에 system 함수의 주소와 ```/bin/sh``` 문자열을 넣어주고 read@plt를 호출하여 셸을 흭득할 것이다.

<br>

근데 문제에서 libc 파일을 제공하였다.

따라서 강의처럼 pwntools를 이용해서 해야 할 것 같다..!

대부분 강의를 보고 코드를 짰고, 마지막 got overwrite 부분은 강의내용을 정리하면 다음과 같다.

read@got를 덮어쓰기 위해 read@plt를 호출할텐데, 인자로 값을 넘겨줘야 하는데 rdx 가젯이 없다.

이 가젯은 libc의 코드 가젯이나, libc_csu_init가젯을 사용하여 해결할 수 있다고 한다.

그러나 이 문제에서는 rdx 값이 매우 크게 설정되어서 필요는 없다고 한다.

--> 이 부분은 강의는 우분투 18.04 버전이고 이 버전에서는 매우 큰 값으로 되있는데, 20.04는 0으로 설정됨

<br>

```python

```





