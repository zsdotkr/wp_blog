멀티쓰레드에서의 락 (Lock)
============

이번에는 리눅스 멀티쓰레드 환경에서 빈번하게 사용되는 pthread 계열 락의 종류 및 각각의 특징과  프로그래머가 상황에 따라 간단하게 아토믹 함수를 이용하여 락을 만드는 방법에 대해서 얘기하고자 합니다.

 락의  주요 사용 목적은 다수의 쓰레드가 특정 자료 영역에 대한 (Read Only 접근을 제외한) **Read & Write** 또는 **Write & Write** 형태의 동시 접근을 차단하여 자료에 **안전**하게 접근할 수 있도록 하는 것이라고 할 수 있고 어떤 방식의 락을 사용해도 이러한 목적은 달성할 수가 있습니다.

하지만 특정 자료 접근을 다수의 쓰레드들이  <u>**순차적**</u>으로  접근하도록 만드는 것이기 때문에 필연적으로 멀티쓰레드 프로그램의 동시 실행성의 저하 (성능 저하) 를 발생시킬 수 밖에 없으므로, 각각의 락 방식에 대한 이해를 통해 **동시 실행성의 저하를 최대한 억제**할 수 있는 방식을 사용할 필요가 있습니다.

여기서는 각각의 락에 대한 기본적인 사용 방법이 아닌 락이 동작하는 방식에 대한 이해도를 넓힐 수 있는 얘기를 하고자 합니다.

-------

spinlock - Busy Waiting
---------

*pthread_spin_xxx* 계열의 함수들이 제공하는 락 방식으로 **Busy Waiting** 형태로 락을 구현하고 사용합니다. 

이해를 돕기 위해 pthread_spin_lock 함수를 C 언어를 이용하여 동작 방식을 표현한다면 아래와 같은 의사 코드를 통해 동작 방식을 이해할 수 있습니다. 

```cpp
int lock_val = 0; // 락 상태를 저장하기 위한 전역 변수 
...
int pthread_spin_lock (int* lock_ptr) {
    while (true) {      // Busy Waiting 
        if ((*locl_ptr) == 0) {
            (*lock_ptr) = 1;
            return 0;
        }
    }
    return EDEADLK;
}
...
void my_function() {
    ...
    pthread_spin_lock(&lock_val); 
    ... // critical section
    pthread_spin_unlock(&lock_val);
}
```

> 위의 코드는 의사 코드이므로 실제는 이보다 좀 더 복잡하며 CPU/OS에 따라 구현 방식에도 차이가 있습니다.

spinlock은 호출자가 락을 획득하기 위해 spinlock을 호출하는 시점부터 락이 얻어질 때까지 SW 제어권이 호출자 (위의 예에서 my_function) 에게 돌아가지 않으며, 무한루프 (**Busy Waiting**) 로 대기를 하기 때문에 **락 획득시까지 CPU 부하를 상승**시킨다는 특징이 있다고 할 수 있습니다.

그러므로 spinlock을 사용하려고 하는 경우에는 상기 예제의 "critical section" 에 해당하는 <u>**영역의 수행 시간이 상당히 짧고 critical section 내에서 blocking I/O 특성을 가지는 OS 호출등을 사용하지 않는 경우**</u>에 적합한 락 방법이라고 할 수 있습니다.

spinlock의 바람직하지 않은 사용 오의 예를 한번 보겠습니다.

> 조건 
> - 쓰레드 두 개 (A, B)를 사용중이다.
> - critical section 내에서 사용자 입력을 받아 저장한다.

> 시나리오 
> - A 쓰레드가 락을 획득한 후 사용자 입력 대기를 위해 getchar() 호출
> - getchar()는 기본적으로 blocking I/O 이므로 스케쥴링 권한이 OS로 이동 
> - OS에서 B 쓰레드로 스케쥴링 전환
> - B 쓰레드에서도 락을 얻기 위해 spinlock 호출
> - spinlock 내에서 무한 루프 대기 수행


> 예상할 수 있는 결과
> - 사용자 입력이 발생해서 OS가 A 쓰레드로 스케쥴링을 전환할 때까지 
> - B 쓰레드가 구동중인 CPU Core는 부하율 100%를 유지
> - Deadlock 상황이 아니지만 의도치 않은 동작 발생 

spinlock이 위와 같은 문제를 야기할 수는 있어 사용 시 주의가 필요하긴 하지만 락을 획득하기 위한 내부 로직의 복잡도나 쓰레드간 절체를 위한 부하가 다른 방식에 비해 상대적으로 작기 때문에 상당히 빈번하게 락을 수행해야 하는 경우에 주로 사용된다.

----------

mutex - context switching if busy (pthread_mutex_ ...)  
---------

Description 

----------

rwlock - conditional context switching if busy (pthread_rwlock_ ...)  
---------

Description 

----------

make your own lock if you need another
---------

Description 

