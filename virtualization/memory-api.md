# 메모리 관리 API

# 메모리 공간의 종류

## 스택

스택은 할당과 반환이 컴파일러에 의해 암묵적으로 이루어지는 메모리 공간이다. 이러한 이유 때문에 때로는 **자동(automatic)** 메모리라고 불린다.

c 프로그램에서 스택에 메모리를 선언하는 것은 쉽다.

```c
void func() {
    int x;
    ...
}
```

컴파일러가 작업을 수행하여 func()가 호출될 때 스택에 공간을 확보한다. 함수에서 리턴하면 컴파일러가 메모리를 반환해준다. 따라서 함수 리턴 후에도 유지되어야 하는 정보는 스택에 저장하지 않는 것이 좋다.

## 힙

오랫동안 값이 유지되어야 하는 변수를 위해 **힙(heap)** 메모리에 저장할 필요가 있다. 모든 할당과 반환이 프로그래머에 의해 명시적으로 이루어져야 하기에 많은 버그의 원인이 되고는 한다. 그러나 조심하고 주의하면 큰 문제 없이 인터페이스를 사용할 수 있다.

```c
void func() {
    int *x = (int *) malloc(sizeof(int));
    ...
}
```

이 코드에는 주의 깊게 볼 사항이 있다. 한 행에 스택과 힙 할당이 모두 발생한다는 것이다.  컴파일러는 포인터 변수의 선언을 만나면 정수 포인터를 위한 공간을 스택에 할당할 것이고, malloc()을 호출하여 정수 공간을 힙으로부터 요구할 것이다.

이러한 명시적 성질과 다양한 쓰임새 때문에 힙 메모리의 사용은 주의깊게 사용할 필요가 있다.

# malloc() 함수

malloc() 호출은 매우 간단하다. 힙에 요청할 공간의 크기를 넘겨주면 공간에 대한 포인터를 반환하거나 실패의 경우 NULL을 반환하게 된다.

```c
#include <stdlib.h>
...
void *malloc(size_t size);
```

malloc의 man 페이지를 보면 알 수 있듯이 호출할 때 size 인자를 넘기게 된다. 이 인자는 필요 공간의 크기를 바이트 단위로 표시한 것인데, 대부분 malloc(10) 처럼 숫자를 넘기지 않는다. 대신 다양한 루틴과 매크로를 활용한다.

```c
double *d = (doube *) malloc(sizeof(double));
```

malloc 호출에서는 정확한 크기의 공간을 요청하기 위해 sizeof 연산자를 호출한다. sizeof는 컴파일 시간 연산자로, 컴파일 중에 숫자(이 경우에는 double이므로 8)로 대체되어 malloc으로 전달된다.

데이터 타입 뿐 아니라 변수의 이름도 sizeof의 인자로 전달할 수 있는데, 원하는 결과가 나오지 않을 수 있으니 조심해야 한다.

```c
int *x = malloc(10 * sizeof(int));
printf("%d\n", sizeof(x));
```

첫번째 행에서는 정수형 원소 10개를 가지는 배열 공간을 선언하였다. 다음 줄에서 sizeof를 사용해보면 4 혹은 8을 반환한다. 이 경우 sizeof()는 동적으로 할당받은 메모리의 크기가 아닌 정수를 가리키는 포인터의 크기가 얼마인지 물어본다고 생각하기 떄문이다. 그러나 다음과 같이 사용하면 기대한 대로 동작하기도 한다.

```c
int x[10];
printf("%d\n", sizeof(x));
```

이 경우에는 변수 x에 40바이트가 할당되었다는 것을 컴파일러가 알 수 있다.

또 조심해야할 경우는 문자열을 다룰 때이다. 문자열을 위한 공간을 선언할 때는 다음과 같이 사용한다. `malloc(strlen(s) + 1);` 이 문장은 문자열의 길이를 얻어낸 뒤 문자열 끝을 나타내는 문자를 위한 공간을 확보하기 위해 1바이트를 더한다. sizeof 사용은 여기서 문제를 일으킬 수 있다.

# free() 함수

할당된 메모리를 해제하는 것을 잊어서는 안된다. 더 이상 사용되지 않는 힙 메모리르 해제하기 위해 free()를 호출한다. free()의 인자는 malloc()에 의해 반환된 포인터를 받는다.

```c
int *x = malloc(10 * sizeof(int));
...
free(x);
```

# 흔한 오류

## 메모리 할당 잊어버리기 

많은 루틴은 자신이 호출되기 전에 피룡한 메모리가 할당되었다고 가정한다. 예를 들어 strcpy(dst, src) 루틴은 소스 포인터에서 목적 포인터로 문자열을 복사한다. 그러나 주의하지 않으면 다음과 같이 코드를 작성할 수 있다.

```c
char *src = "hello";
char *dst;
strcpy(dst, src);
```

이 코드를 실행하면 **세그멘테이션 폴트(segmentaion fault)** 가 발생할 가능성이 높다. 따라서 다음과 같이 메모리를 할당해주어야 한다.

```c
char *src = "hello";
char *dst = (char *) malloc(strlen(src) + 1);
strcpy(dst, src);
```

또는 strdup()을 이용하면 훨씬 편하게 할 수 있다. 만약 할당할 메모리가 부족할 시 NULL을 반환할 수 있다.

```c
char *src = "hello";
char *dst = strdup(src);
```

## 메모리 부족하게 할당받기

이 오류는 메모리를 부족하게 할당받는 것으로 때때로 버퍼 오버플로우(buffer overflow) 라고 불린다. 목적지 버퍼의 공간을 약간 부족하게 받을 때 일어난다.

```c
char *src = "hello";
char *dst = (char *) malloc(strlen(src)); // 1 바이트 부족
strcpy(dst, src);
```

상황에 따라서는 올바르게 동작하는 것처럼 보일 수도 있지만, 이는 매우 유해할 수 있고 보안 문제를 일으킬 수도 있다.

## 할당받은 메모리 초기화하지 않기

이 오류의 경우 malloc()을 호출하였지만 새로 할당받은 데이터 타입에 특정 값을 넣지 않는 것이다. 초기화 하지 않는다면 프로그램은 **초기화되지 않은 읽기(uninitialized read)** , 즉 힙으로부터 알 수 없는 값을 읽을 수 있다.

## 메모리 해제하지 않기

일반적인 오류로 **메모리 누수(memory leak)** 다. 메모리 해제를 잊었을 때 장시간 실행되는 응용 프로그램이나 운영체제와 같은 시스템 프로그램에게는 치명적이다. 메모리 누수가 발생하면 결국 메모리가 부족해지고 시스템을 재시작 해야한다.

경우에 따라서 free()를 호출하지 않는게 타당하게 보이는 경우가 있다. 짧게 실행되는 프로그램의 경우 프로세스가 종료될 때 OS가 메모리를 회수해 가기 때문에 메모리 누수는 일어나지 않을 것이다. 하지만 장기적인 관점에서 이는 들여서는 안되는 버릇으로 명시적으로 할당받은 메모리는 꼭 해제하는 습관을 들여야한다.

## 메모리 사용이 끝나기 전에 메모리 해제하기

때로는 메모리 사용이 끝나기 전에 메모리를 해제한다. 이는 **dangling pointer**라고 불리며 심각한 실수이다. 차후에 그 포인터를 사용하면 유효한 값을 덮어쓰거나 크래시가 날 것이다.

## 반복적으로 메모리 해제하기
가끔 프로그램이 메모리를 두번 이상 해제하는 경우가 있다. 이를 **이중 해제(double free)** 라고 한다. 결과는 예측하기가 매우 어려운데, 서로 다른 변수가 한 공간을 공유하거나 크래시가 일어나는 등 문제가 발생한다.

## free() 잘못 호출하기

free()는 malloc()으로 받은 포인터를 전달 받을 것으로 예상한다. 그 이외의 값을 전달할 경우 **유효하지 않은 해제(invalid frees)** 가 발생할 것이다.