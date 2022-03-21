# 세마포어

# 세마포어: 정의

세마포어는 정수값을 갖는 객체로서 2개의 루틴으로 조작할 수 있다. POSIX 표준에서 이들은 sem_wiat()와 sem_post()이다. 세마포어를 사용하기 전 "제일 먼저" 값을 초기화 해야한다.

```c
#include <semaphore.h>
sem_t s;
sem_init(&s, 0, 1);
```
각 인자는 세마포어 포인터, 프로세스 공유 여부, 초기값을 의미한다. 하지만 2번째 인자의 경우 LinuxThreads는 세마포어가 여러 프로세스 간에 공유하는 것을 지원하지 않기 떄문에 0이 아닐 경우 ENOSYS 에러코드를 반환한다.

초기화 후 sem_wait()과 sem_post()를 호출하여 세마포어를 제어할 수 있다.

```c
int sem_wait(sem_t *s) {
    // semaphore s의 값을 1 감소시킨다.
    // s의 값이 음수이면 wait한다.
}

int sem_post(sem_t *s) {
    // semaphore s의 값을 1 증가시킨다.
    // 하나 이상의 쓰레드가 기다리고 있으면 하나를 깨운다.
}
```

이 두 루틴의 중요한 성질을 알 필요가 있다.
## sem_wait()과 sem_post()의 성질
- 세마포어가 음수라면 그 값은 현재 대기 중인 쓰레드의 개수와 같다.
- 이 두 개의 함수는 원자적으로 실행된다.

### sem_wait()
- 세마포어의 값이 1 이상이면 즉시 리턴한다.
- 혹은 세마포어의 값이 1 이상이 될 때까지 호출자를 대기 시킨다.
- 다수의 쓰레드가 호출할 수 있기에 대기큐에는 다수의 쓰레드가 존재할 수 있다.
  
### sem_post()
- wait과 달리 대기하지 않는다.
- 세마포어의 값을 증가시키고 대기 중인 쓰레드 중 하나를 깨운다.

# 이진 세마포어 (락)

락을 사용하듯이 세마포어를 사용하려면 다음과 같이 하면 된다.

```c
sem_t m;
sem_init(&m, 0, 1);
sem_wait(&m);

//임계 영역 코드

sem_post(&m);
```

락은 두 개의 상태(사용 가능, 사용 중)만 존재하므로 **이진 세마포어(binary semaphore)** 라고도 불린다.

# 순서 보장을 위한 세마포어

이전에 **컨디션 변수**를 사용했던 것과 유사하게 세마포어를 순서를 정하기 위한 도구로 사용할 수 있다.

```c
sem_t s;

void *child(void *arg) {
    printf("child\n");
    sem_post(&s); // signal here: child is done
    return NULL;
}

int main(int argc, char *argv[]) {
    sem_init(&s, 0, 0);
    printf("parent: begin\n");
    pthread_t c;
    Pthread_create(&c, NULL, child, NULL);
    sem_wait(&s); // wait here for child
    printf("parent: end\n");
    return 0;
}
```
