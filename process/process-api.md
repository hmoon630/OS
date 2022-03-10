# 프로세스 API

UNIX 시스템에서의 프로세스 생성에 대해서 다룬다. UNIX에서는 프로세스를 생성하기 위해 fork()와 exec() 시스템 콜을 사용하고, wait()이라는 자신이 생성한 프로세스가 종료되기를 대기하는 콜도 존재한다.

# fork()
fork는 프로세스를 생성하기 위한 시스템 콜이다. fork를 호출한 프로세스는 부모 프로세스가 되고, fork에 의해 생성된 프로세스는 부모 프로세스의 메모리를 복사하고 자식 프로세스가 된다.

fork 시스템 콜을 사용한 간단한 예제 코드이다.
```c
#include <stdio.h>
#include <unistd.h>
 
int main() {
    int x = 0;
     
    fork();
     
    x = 1;
    printf("PID : %ld,  x : %d\n", getpid(), x);

    return 0;
}
```
이를 실행시키면 다음과 같은 결과가 출력될 것이다.
`
PID : 43889,  x : 1
`
`
PID : 43895,  x : 1 
`

부모 프로세스와 자식 프로세스를 구분짓고 싶은 경우 fork의 반환 값을 이용해 구분할 수 있다. fork는 실패시 -1을 반환하고, 자식일 경우에는 0, 부모일 경우 자식 프로세스의 pid를 반환한다.

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
 
int main(int argc, char *argv[]) {
     
    pid_t pid;
     
    int x = 0;
     
    pid = fork();
 
    if(pid < 0) {  // fork 실패 시
        printf("fork Fail! \n");
        return -1;
    }
    else if(pid == 0){  // 자식 프로세스
        x = 1;
        printf("자식 PID : %ld,  x : %d\n", (long)getpid(), x);
    }
    else {  // 부모 프로세스
        x = 2;
        printf("부모 PID : %ld,  x : %d , pid : %d\n", (long)getpid(), x, pid);
    }
     
    return 0;
 
}
```
위의 실행결과는 다음과 같다.

`
부모 PID : 46834,  x : 2 , pid : 46838
`

`
자식 PID : 46838,  x : 1
`
하지만 위의 실행결과는 매번 다를 수 있다. CPU 스케줄러가 실행할 프로세스를 선택하는 과정이 복잡하고 상황에 따라 달라지기 때문에 쉽게 어느 프로세스가 먼저 실행될 지 알 수 없다.

# wait()

부모 프로세스가 자식 프로세스를 기다릴 때 호출하는 시스템 콜이다. wait 함수의 인자로 *wstatus를 넘겨주면 종료 상태를 알 수 있다. 이 wstatus 값은 매크로를 사용하여 상태를 알 수 있다.

위의 코드에서 wait() 콜을 추가한다면 이렇게 될 것이다.
```c
...
    else {  // 부모 프로세스
        wait(NULL)
        x = 2;
        printf("부모 PID : %ld,  x : %d , pid : %d\n", (long)getpid(), x, pid);
    }
...
```
이렇게 할 경우 부모 프로세스는 자식 프로세스가 끝날 때 까지 기다리기 때문에 아래와 같은 순서가 보장된다.

`
부모 PID : 46834,  x : 2 , pid : 46838
`

`
자식 PID : 46838,  x : 1
`

# exec()

exec()은 자기 자신을 복제하는 것이 아닌 다른 프로그램을 실행할 때 사용한다. exec의 경우 변형된 함수가 많은데, 그 이름에서 활용방법을 알 수 있다.

| 알파벳 | 기능 |
|-------|------|
|p      |환경 변수 PATH에 있는 명령인지 확인|
|e      |인자에 char *argv[] 형태로 환경 변수를 전달|
|v      |인자를 char *argv[] 형태로 전달|
|l      |인자를 가변인자로 전달|

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    int pid = fork();
    if (pid == -1) { //fork 실패시
        fprintf(stderr, "fork fail");
        exit(1);
    } else if (pid == 0) {
        printf("It's child process");
        execlp("ls", "-al");
        printf("This line will never be reached");
    } else {
        wait(NULL);
        // 이후에 진행될 코드
    }

}
```
위의 프로그램이 정상적으로 실행될 경우 execlp()의 밑으로는 더이상 실행되지 않는데, 이는 exec() 시스템 콜의 특성 때문이다. 

exec을 실행시키면 다음과 같은 일이 일어난다.
- 인자가 주어지면 코드와 정적 데이터를 읽어 들여 현재 실행 중인 프로세스의 코드 세크먼트와 정적 데이터를 덮어 쓴다.
- 힙과 스택, 다른 주소 공간도 새로운 프로그램을 위해 초기화 된다.
- 프로세스의 argv와 같은 인자를 전달하여 프로그램을 실행시킨다.

새로운 프로세스를 생성하지 않고 실행할 프로그램으로 대체하기 때문에 exec의 밑으로는 더 이상 실행 되지 않는다.
