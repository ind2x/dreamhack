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



