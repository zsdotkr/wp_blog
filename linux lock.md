리눅스 멀티쓰레드에서의 락 (Lock)
============

이번에는 멀티쓰레드 환경에서 빈번하게 사용되는 pthread 계열 락의 종류 및 각각의 특징과  프로그래머가 상황에 따라 간단하게 아토믹 함수를 이용하여 락을 만드는 방법에 대해서 얘기하고자 합니다.

락의  주요 사용 목적은 다수의 쓰레드가 특정 자료 영역에 대해서 Read Only 접근을 제외한 **Read & Write** 또는 **Write & Write** 형태의 동시 접근을 차단하여 자료에 **안전**하게 접근할 수 있도록 하는 것이라고 할 수 있고 어떤 방식의 락을 사용해도 이러한 목적은 달성할 수가 있습니다.

하지만 특정 자료 접근을 다수의 쓰레드들이  <u>**순차적**</u>으로  접근하도록 만드는 것이기 때문에 필연적으로 멀티쓰레드 프로그램의 동시 실행성의 저하 (성능 저하) 를 발생시킬 수 밖에 없으므로, 각각의 락 방식에 대한 이해를 통해 **동시 실행성의 저하를 최대한 억제**할 수 있는 방식을 사용할 필요가 있습니다.

여기서는 각각의 락에 대한 기본적인 사용 방법이 아닌 락이 동작하는 방식에 대한 이해도를 넓혀서 상황에 따른 적절한 락 방식을 선택할 때 도움을 주고자 합니다.

-------

spin / spinlock - Busy Waiting
---------

*pthread_spin_xxx* 계열의 함수들을 통해 제공하는 락 방식으로 **Busy Waiting** 형태로 락을 구현합니다. 

이해를 돕기 위해 pthread_spin_lock 함수를 C 언어 형태로 동작 방식을 이해할 수 있도록 표현한다면 아래와 같이 작성을 할 수 있습니다.

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
int pthread_spin_unlock(int* lock_ptr) {
    while(true) {   // Busy Waiting
        if ((*lock_ptr) == 1) {
            (*lock_ptr) = 0;
            return 0;
        }
    }
    return -1;
}
...
void my_function() {
    ...
    pthread_spin_lock(&lock_val);   // lock_val 변수값을 0 -> 1로 변경 
    ... // critical section
    pthread_spin_unlock(&lock_val); // lock_val 변수값을 1 -> 0로 변경 
}
```

> 위의 코드는 의사 코드이므로 실제는 이보다 좀 더 복잡하며 CPU/OS에 따라 구현 방식에도 조금씩 차이가 있습니다만 OS가 개입하지 않고 락의 획득 및 반환이 이루어진다는 구현 방식의 동일성을 가지고 있습니다.

spinlock은 호출자가 락을 획득하기 위해 spinlock을 호출하는 시점부터 락이 얻어질 때까지 SW 제어권이 호출자 (위의 예에서 my_function) 에게 돌아가지 않으며, 무한루프 (**Busy Waiting**) 로 대기를 하기 때문에 **락 획득시까지 CPU 부하를 상승**시킨다는 특징이 있습니다. 

그러므로 spinlock을 사용하려고 하는 경우에는 상기 예제의 "critical section" 에 해당하는 영역의 <u>**수행 시간이 상당히 짧고 수행 시간이 일정하며 (deterministic)  critical section 내에서 blocking I/O 특성을 가지는 OS 호출등을 사용하지 않는 경우**</u>에는 제일 효율적인 방법이라고 할 수 있습니다.

spinlock의 바람직하지 않은 사용 예를 한번 보겠습니다.

> 조건 
> - 쓰레드 두 개 (A, B)를 사용중이다.
> - critical section 에는 사용자 입력을 받아 저장하는 로직이 있다. 

> 시나리오 
> - A 쓰레드가 락을 획득한 후 사용자 입력 대기를 위해 getchar() 호출
> - getchar()는 기본적으로 blocking I/O 이므로 A 쓰레드는 Sleep/Waiting 상태로 전환
> - OS 스케쥴러가 B 쓰레드를 Run 상태로 전환
> - B 쓰레드에서 락을 얻기 위해 spinlock을 호출하나 A 쓰레드가 이미 점유중이므로 무한루프 대기 수행


> 예상 결과
> - 사용자 입력이 발생해서 OS가 A 쓰레드를 Run 상태로 전환할 때까지 
> - B 쓰레드가 구동중인 CPU Core는 부하율 100%를 유지

spinlock이 위와 같은 문제를 야기할 수는 있어 사용 시 주의가 필요하긴 하지만 락을 획득하기 위한 내부 로직의 복잡도나 쓰레드간 절체를 위한 부하가 다른 방식에 비해 상대적으로 작기 때문에 (OS의 개입이 없기 때문에 가능한 부분) 상당히 빈번하게 락을 수행해야 하는 경우에 주로 사용됩니다. 
예를 들면 쓰레드간에 데이타/이벤트등의 전달을 위해 사용하는 FIFO, LIFO, Queue 등의 구조에 대한 동시 접근 제어가 필요한 경우에 유용하다고 할 수 있습니다. 

----------

mutex - context switching if already locked
---------

이제부터 설명하는 pthread 계열 락 함수들은 처음에 설명한 spinlock과는 다르게 락의 획득과 반환 과정에 모두 OS가 개입하는 형태입니다.

일단 OS가 개입되면 락의 획득을 위한 다양한 조건 설정이 가능해 집니다. 예를 들어 획득까지의 대기 시간을 설정할 수도 있으며 대기중인 동안 CPU를 다른 쓰레드가 사용할 수 있도록 넘길 수도 있습니다. (spinlock은 이런 것들이 불가능합니다)

이번에 설명할 mutex가 위와 같은 다양한 조건으로 락을 거는 것이 가능하며 pthread에서 *pthread_mutex_xxx*  함수들을 통해 제공됩니다.
실제 mutex를 구현하는 내부 로직은 커널에 대한 이해를 포함하여 기술적으로 상당히 어려운 내용들이 많기는 하지만 spinlock과의 차이를 확인하기 위한 정도로 간단하게 C 언어 형태로 동작 방식을 표현한다면 아래와 같이 작성을 할 수 있습니다.

```cpp
int lock_val = 0; // 락 상태를 저장하기 위한 전역 변수
...
int pthread_mutex_lock (int* lock_ptr) {
    if ((*lock_ptr) == 0) {
        (*lock_ptr) = 1;    
        return 0;
    }
    else {
        errno = wakeup_me_at_my_turn (); // 쓰레드가 Sleep 상태로 전환되는 지점
        (*lock_ptr) = 1;    // send_signal_to_wakeup_others에 의해 다시 깨어날 때 수행되는 지점 
        return 0;
    }
}
int pthread_mutex_unlock(int* lock_ptr) {
    if ((*lock_ptr) == 1) {
        (*lock_ptr) = 0;
        send_signal_to_wakeup_others(); // 이 함수 호출 시 Sleep 되지는 않습니다.
        return 0; 
    }
    errno = -1; 
    return 0; 
}
...
void my_function() {
    ...
    pthread_mutex_lock(&lock_val);   
    ... // critical section
    pthread_mutex_unlock(&lock_val);
}

```
> 구체적으로 표현하기에는 너무 복잡해서 복잡한 부분은 아래와 같이 기능을 표현하기 위한 두개의 함수로 단순하게 표현해 봤습니다. 

> wakeup_me_at_my_turn :  OS가 나중에 스케쥴링을 할 수 있도록 mutex 구조체내의 FIFO 리스트에 쓰레드 정보를 추가해서 OS로 전달한 후 자신은 Sleep 상태로 전환되서 CPU를 다른 쓰레드/프로그램이 사용할 수 있도록 OS에게 리스케쥴링을 요청하는 과정이 포함되어 있습니다.

> send_signal_to_wakeup_others :  대기중이던 쓰레드들이 락을 획득할 수 있도록 (mutex내의 FIFO 리스트에 등록된 순서대로) OS에게 이벤트를 발생시킵니다. 함수 호출 시 쓰레드는 Sleep 상태로 빠지지는 않으며, OS가 Sleep 상태인 쓰레드를 동시 수행이 가능하다고 판단이 되며 대기중인 쓰레드는 활성화 시킬 수 있습니다.


mutex는 spinlock의 Busy Waiting 과는 다르게 락을 획득할 수 없다면 SW 제어권이 OS로 전환되므로 **Busy Waiting이 발생하지 않는다**는 특징이 있습니다.

또한 spinlock의 경우에는 락 획득을 대기중이던 쓰레드들 중 어느 쓰레드가 락을 획득할 것인지 순서를 특정할 수 없는 반면 mutex를 사용하게 되면 mutex 내부의 FIFO 리스트를 이용하여 락 획득 대기 상태로 상태가 변경된 순서대로 재 획득이 이루어지게 됩니다. 이러한 일종의 Fairness 형태를 제공하는 대기열 관리 방식은 자칫 잘못하면 일부  쓰레드들이 락을 독점해서 **critical section의 수행이 일부 쓰레드들로 편중되는 현상을 막을 수 있다**는 장점이 있습니다.

이 외에도 *pthread_mutex_timed_lock*을 이용하여 일정 시간내에 락을 확득할 수 없는 경우에는 락의 획득을 포기하도록 할 수 있고 (대기 시간에는 OS가 리스케쥴링을 수행하고 타임아웃 처리 역시 OS에서 수행됨) 락에 대한 **재귀 호출** (동일 쓰레드내의 재귀함수내에서 락을 반복적으로 호출하는 경우)도 제공합니다.  

mutex는 위와 같은 **CPU 점유 시간의 최소화 / Fairness / 재귀호출 / 타임아웃**과 같이 락을 획득하는 방법을 좀 더 다양하게 할 수 있는 장점이 있어서  범용적으로 제일 많이 사용하는 방법이지만, 이런 특징을 제공하기 위해서 spinlock에 비해 락의 획득을 위해 소요되는 오버헤드가 클 뿐만 아니라 OS에서의 리스케쥴링 및 타이머 관리등을 위한 오버헤드도 있습니다. 

프로그램내에서 spinlock과 mutex 중 어느 형태를 사용하는 것이 적합할 것인가는 **락킹의 횟수** 및 **critical section 의 수행 시간**, **락을 공유하는 쓰레드 수**등 여러가지 변수가 있어 초기 구현시에 판단하기가 쉽지 않을 수 있으므로 초기에는 mutex를 이용하도록 구현을 한 후에 선택적으로 spinlock을 적용하는 것도 좋은 구현 절차일 수 있습니다.


----------

rwlock - conditional context switching 
---------

이번에 설명할 것은 rwlock으로 pthread에서는 *pthread_rwlock_xxx* 계열의 함수들을 통해 제공됩니다. 

기본적으로는 mutex와 동일하게  CPU 점유 시간의 최소화 / Fairness / 재귀호출 / 타임아웃을의 조건으로 락을 획득할 수 있으며, mutex에서는  critical section내에서의 자료 접근이 Read or Write 형태를 구분하지 않지만 rwlock에서는 **Read Only or Read/Write**로 구분하여 락의 획득을 좀 더 최적화한다는 특징이 있습니다. 

그럼 일단 mutex와의 차이를 확인할 수 있도록 간단하게 C 언어 형태로 동작 방식을 표현해 보도록 하겠습니다.

```cpp
int lock_val = 0;       // 락 상태를 저장하기 위한 전역 변수
int lock_reader = 0;    // rwlock 의 reader 수를 저장하기 위한 변수 
...
int pthread_rwlock_rdlock (int* lock_ptr, int* reader) {
    if ((*lock_ptr) == 0) { 
        (*lock_ptr) = 1;   
        (*reader) = 1;    
    }
    else {  // if already locked 
        if ((*reader) > 0) {    // if locked by reader 
            (*reader) ++;       // Just increase reader count 
        }
        else {                  // if locked by writer 
            errno = wakeup_me_at_my_turn_read (); 
            (*lock_ptr) = 1;
            return 0;
        }
    }
    return 0;
}

int pthread_rwlock_wrlock (int* lock_ptr, int* reader) {
    if ((*lock_ptr) == 0) { 
        (*lock_ptr) = 1;   
        (*reader) = 1;    
        return 0;
    }
    else {  
        errno = wakeup_me_at_my_turn_write (); 
        (*lock_ptr) = 1;
        return 0;
    }
}

int pthread_rwlock_unlock (int* lock_ptr, int* reader) {
    if ((*lock_ptr) == 1) {
        if ((*reader) > 0)  {   
            (*reader)--; 
            return 0;
        }
    
        (*lock_ptr) = 0;
        if (send_signal_to_wakeup_writer() == 0) {
            send_signal_to_wakeup_reader();
        }
        return 0; 
    }
    errno = -1; 
    return 0; 
}
...
void my_function() {
    ...
    pthread_rwlock_rdlock(&lock_val, &lock_reader);   
    ... // critical section
    pthread_rwlock_unlock(&lock_val, &lock_reader);
}
```
> mutex와 비교할 수 있도록 가급적 mutex와 비슷한 형태를 가지도록 표현을 했습니다.  

> wakeup_me_at_my_turn_xxx :  앞서 설명했던 mutex의 wakeup_me_at_my_turn와 동일한 기능을 수행합니다. 단, rwlock의 내부에는 reader와 writer로 구분된 두가지의 FIFO가 있어 호출하는 함수에 따라 큐잉되는 FIFO의 종류가 달라집니다.

> send_signal_to_wakeup_wrter_first :  muterx의 함수와 유사하게 대기중이던 쓰레드들이 락을 획득할 수 있도록 OS에게 이벤트를 발생시킵니다. 단, reader FIFO와 writer FIFO 중 writer FIFO에 대기중인 쓰레들을 우선적으로 활성화시키도록 합니다. 

rwlock의 특징은 대기열이 reader와 writer로 구분된다는 것 외에 위의 pthread_rwlock_rdlock() 함수에 보이는 것처럼 **락이 걸려 있는 상태지만 reader가 락을 건 상태(reaer > 0) 라면** Sleep 상태로 전환하지 않고 락을 획득한 상태로 처리한다는 점입니다. 

이런 조건을 추가되게 되면 pthread_rwlock_rdlock()을 호출하는 하나 이상의 쓰레드는 동시에 critical section내의 코드를 수행하는 게 가능해 지므로 모든 쓰레드를 직렬화하는 mutex에 비해 쓰레드들의 동시 실행성의 저하가 훨씬 개선될 수 있게 됩니다. 

물론 다수의 쓰레드가 critical section내의 자원을 안전하게 동시 접근을 하기 위해서는 자원을 접근하는 형태가 자원을 변경하는 형태(write)가 아 아닌 참조만 하는 형태 (read)이기 때문에 함수의 이름 역시 rdlock(read-lock)이라고 붙여지게 된 것입니다.

이런 특징을 잘 활용할 수 있는 코드의 예라면 데이타베이스와 같이 자료를 생성하는 생성자가 하나 또는 수개 이하로 한정되어 있고, 자원을 참조하는 소비자는 불특정의 다수인 경우라고 할 수 있습니다.

----------

make your own lock if you need another
---------

Description 

