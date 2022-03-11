# 제한적 직접 실행

OS는 CPU를 가상화하여 여러 프로세스가 동시에 실행되는 것처럼 보이도록 한다. 기본적인 아이디어는 한 프로세스를 잠깐 실행하고 다른 프로세스를 또 잠깐 실행한다. 이 방식으로 CPU의 시간을 나누어 써 가상화를 구현한다.


# 기본 원리: 제한적 직접 실행

OS 개발자들은 프로그램 실행을 빠르게 하기 위해 **제한적 직접 실행**이라는 기법을 개발했다. 여기서 "직접 실행"은 CPU 상에서 프로그램을 직접 실행시키는 것을 의미한다. 

하지만 단순히 CPU에서 실행할 경우에는 OS가 프로그램이 하는 일에 대해 통제할 수 없고, CPU 가상화를 위한 시분할 기법을 사용하기 위해 프로세스를 쉽사리 중단 시킬 수도 없다.

이 문제들을 해결하기 위해 "제한적" 직접 실행이라는 기법을 개발해 낸 것이다.

# 문제점 1: 제한된 연산

직접 실행은 빠르게 실행된다는 장점이 있다. 하지만 프로세스가 시스템 자원을 요청하는 등 특정 연산을 요청했을 때 문제가 발생하게 된다.

이를 위해 OS에는 2가지의 모드가 도입되었다.
- **사용자 모드(user mode)**: 실행되는 코드의 일들이 제한된다.
- **커널 모드(kernel mode)**: OS의 중요한 코드들이 실행된다. 사용자 모드와 달리 모든 작업을 수행할 수 있다.

사용자 모드는 특권 명령어를 실행하기 위해서는 **시스템 콜**을 사용하면 된다. 시스템 콜은 사용자 프로세스가 하드웨어에게 자원을 요청할 수 있도록 하는 기능이다.

시스템 콜을 실행하면 **trap**이 발생하게 된다. 시스템 콜은 아래의 순서에 따라 실행된다.

1. 사용자 모드에서 system call trap 을 호출한다.
2. 하드웨어가 kernel stack에 register 값들을 저장하고 커널 모드로 이동하여 trap handler로 점프한다.
3. 커널 모드에서 trap을 제어하고 system call을 수행한 뒤 return-from-trap을 호출하며 다시 돌아간다.

# 문제점 2: 프로세스 간 전환

직접 실행의 2번째 문제점은 다른 프로세스로의 전환이 쉽지 않다는 것이다. 그렇다면 어떻게 프로세스로부터 CPU를 돌려받을 수 있을까? 다음에서 2가지 방법을 다룬다.

## 협조 방식: 시스템 콜 호출까지 대기
협조 방식은 프로세스가 주기적으로 CPU 자원을 반환해 줄 것으로 가정하고 만들어진 방식이다. 자원을 반환하기 위해서는 운영체제에게 제어권을 넘겨야 하는데, 시스템 콜을 호출하거나 **트랩**을 발동시켰을 때 OS가 제어권을 얻게 된다.

하지만 이 방식의 스케줄링은 수동적인데, 프로세스에서 시스템 콜을 호출하거나 트랩될 때 까지는 계속 기다려야만 한다. 어떤 이유에서든 프로세스에서 무한 루프가 일어날 경우 문제가 생긴다. 이를 해결하기 위해서는 재부팅하는 방법 밖에는 없다.

## 비협조 방식: 운영체제가 제어권 확보

비협조 방식은 시스템 콜을 호출하지 않더라도 OS가 제어권을 얻을 수 있다. 이는 하드웨어로부터 도움을 받는데, **타이머 인터럽트(timer interrupt)**를 이용하는 것이다. 타이머는 수ms마다 인터럽트라고 불리는 하드웨어 신호를 발생시키고, 운영체제가 interrupt handler로 처리하도록 하는 것이다.

## 문맥의 저장과 복원

시스템 콜이나 인터럽트, 혹은 다른 이유로라도 OS가 제어권을 확보하면 다시 어떤 프로세스를 실행할 지 결정하는 **스케줄링(scheduling)**하게 된다. 이때 기존에 실행하던 A를 계속 실행할 지, 혹은 다른 프로세스 B를 실행할 지 결정하는 것이다.

만약 다른 프로세스를 실행하기로 결정했다면 **문맥 교환(context switch)**을 하게 된다. 실행 중이던 프로세스의 문맥(레지스터 값들)을 메모리에 저장하고 새로운 프로세스의 문맥을 CPU로 읽어들이는 것이다. 문맥 교환을 마치고 나면 return-from-trap을 호출하여 마치 새로운 프로세스가 trap을 호출 하였던 것 처럼 보이게 된다. 

## 만약 인터럽트가 겹친다면?

OS가 시스템 콜을 처리하는 도중에 인터럽트가 발생하는 경우가 발생할 수 있다. 간단한 해법으로는 인터럽트를 처리하는 동안 다른 인터럽트를 **불능화**하는 방법이 있다. 다만 이는 주의할 필요가 있는데, 인터럽트를 장기간 불능화 하게되면 손실되는 인터럽트가 많아진다는 것이다.