# 쓰레드 API

# 쓰레드 생성

멀티 쓰레드 프로그램을 작성 시, 가장 먼저할 일은 새로운 쓰레드의 생성이다. 쓰레드는 해당 인터페이스로 만든다.

```c
#include <pthread.h>
int
pthread_create(pthread_t            *thread,
            const pthread_attr_t    *attr,
            void                    *(*start_routine)(void*),
            void                    *arg);
```

# 쓰레드 종료

다른 쓰레드가 작업을 완료할 때 까지 기다려야 한다면 뭔가 특별한 조치가 필요하다. POSIX 쓰레드에서는 pthread_join()을 부르는 것이다.

```c
int pthread_join(pthread_t thread, void **value_ptr);
```

이 루틴은 2가지 인자를 받는다. pthread_t 는 어떤 쓰레드를 기다리려고 하는지 명시한다.
value_ptr은 쓰레드의 리턴 값을 받는다. 반환 값이 필요 없다면 NULL을 넘겨도 된다.

마지막으로 pthread_create()한 다음 직후에 pthread_join()을 호출한다는 것은 쓰레드를 생성하는 이상한 방법이다. 보통 **프로시저 호출(procedures call)** 이라고 부른다.

# 락

POSIX 쓰레드 라이브러리가 제공하는 가장 유용한 함수는 **락(lock)** 을 통한 임계 영역에 대한 상호 배제 기법이다. 

가장 기본적인 루틴은 다음과 같이 제공된다.

```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

락은 **임계 영역(critical section)** 의 올바른 연산을 보장하기 위해 보호할 때 매우 유용하다. 아마도 다음과 같이 코드를 작성해 볼수 있겠다.

```c
pthread_mutex_t lock;
pthread_mutex_lock(&lock);
x = x + 1;
pthread_mutex_unlock(&lock);
```

사실 위의 코드에는 2가지 문제점이 있다.

하나는 POSIX 쓰레드를 사용할 때는 락을 초기화 하고 사용해야 한다는 것이다. 락을 초기화 하는 방법이 2가지 있다. 정적인 방법으로 PTHREAD_MUTEX_INITIALIZER를 사용하는 것과 동적인 방법으로 pthread_mutex_init()을 호출하는 방법이 있다.

```c
// 정적
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

// 동적
int rc = pthread_mutex_init(&lock, NULL);
assert(rc == 0); // 성공했는지 꼭 확인해야 한다!
```

정적인 방법은 락을 디폴트 값으로 설정한다. 동적인 방법의 루틴의 첫번째 인자는 락 자체의 주소이고, 두 번째 인자는 선택 가능한 속성이다. NULL은 기본값을 의미한다. 락 사용이 끝났다면 초기화에 상응하는 pthread_mutex_destroy()를 호출해야 한다.

두번째 문제는 락과 언락을 호출할 때 에러 코드를 확인하지 않는다는 것이다. UNIX 시스템에서 호출하는 거의 모든 라이브러리 루틴과 마찬가지로 이 루틴들도 실패할 수 있다. 이 경우 여러 쓰레드가 동시에 임계 영역에 들어갈 수 있다.

락과 언락 루틴 외에 pthread 라이브러리에 추가 루틴들어 더 있다.

```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *mutex,
                            struct timespec *abs_timeout);
```

이 두 함수를 눈여겨 볼만하다. trylock은 락이 이미 사용중이라면 실패 코드를 반환한다. timedlock은 타임아웃이 끝나거나 락을 획득하거나의 두 조건 중 하나가 발생하면 리턴한다. 이 두 함수는 사용하지 않는 것이 좋지만, 락 획득 루틴에서 무한정 대기하는 상황(교착 상태)을 피하기 위해 사용되기도 한다.

# 컨디션 변수

POSIX 쓰레드의 경우 주요한 구성 요소로 **컨디션 변수(condition variable)** 가 있다.

```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_signal(pthread_cond_t *cond);
```

이것들이 기본 루틴이다. 컨디션 변수 사용을 위해서는 컨디션 변수와 연결된 락이 "반드시" 존재해야 한다.

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALZIER;

Pthread_mutex_lock(&lock);
while (ready == 0)
    Pthread_cond_wait(&cond, &lock);
Pthread_mutex_unlock(&lock);
```

이 코드는 락과 컨디션 변수를 초기화 한 후 ready가 0이 아니면 다른 쓰레드가 깨워줄 때까지 대기 루틴을 호출한다.

다른 쓰레드에서 잠자는 쓰레드를 꺠우는 코드는 다음과 같다.

```c
Pthread_mutex_lock(&lock);
ready = 1;
Pthread_cond_signal(&cond);
Pthread_mutex_unlock(&lock);
```

이 코드에서 유의할 점이 있다. 첫째 시그널을 보내고 전역 변수 ready를 수정할 때 반드시 락을 가지고 있어야 한다. 이는 경쟁 조건이 발생하지 않는다는 것을 보장한다.

두번째 시그널 대기 함수에서는 락을 두 번째 인자로 받고 있지만, 시그널 보내기 함수에서는 조건만을 인자로 받는다. 이는 시그널 대기 함수는 쓰레드를 재우는 것 외에도 락을 반납하여야 하기 때문이다. 만약 반납하지 않을 경우 다른 쓰레드가 깨우기 위한 시그널을 보내지 못할 것이다.

마지막으로 대기하는 쓰레드가 조건을 검사할 때 if문 대신 while문을 사용한다는 것이다. 만약 변수를 제대로 갱신하지 않고 대기하던 쓰레드를 꺠울 수 있는데, 이 때 재검사를 하지 않으면 변경되지 않고도 계속 실행 되어버릴 것이다. 시그널은 변경된 것 같으니 검사해보라는 힌트 정도로 생각하는 것이 좋다.
