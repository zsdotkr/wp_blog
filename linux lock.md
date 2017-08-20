리눅스 멀티쓰레드에서의 락 (Pthread Lock)
============

이번에는 멀티쓰레드 환경에서 빈번하게 사용되는 POSIX thread 라이브러리인 pthread내에 정의된 락의 종류 및 각각의 특징을 살표보고자 합니다. 

락의  주요 사용 목적은 다수의 쓰레드가 특정 영역 (자료 또는 자원)에 대해서 Read Only 접근을 제외한 **Read & Write** 또는 **Write & Write** 와 같이 혼합된 형태로 동시 접근을 하는 것을 차단해서 자료/자원이 **안전**하게 (자료/자원의 변경 또는 사용이 완료되기 전에는 다른 쓰레드가 동일한 자료/자원를 접근하지 못하도록 차단해서) 처리될 수 있도록 하는 것이라고 할 수 있으며 어떤 방식의 락을 사용해도 이러한 목적은 달성할 수가 있습니다.

하지만 이런 안정성은 특정 자료/자원에 대한 접근을 요청하는 다수의 쓰레드들을  <u>**순차적**</u>으로  접근하도록 만드는 것이기 때문에 필연적으로 멀티쓰레드 프로그램의 동시 실행성의 저하 (성능 저하) 를 발생시킬 수 밖에 없습니다.

여기서는 각각의 락에 대한 사용 방법보다는 락이 동작하는 방식에 대한 이해를 통해 **동시 실행성의 저하를 최소화**할 수 있도록 상황에 따른 적절한 락 방식의 선택에 도움을 주고자 합니다.

-------

spinlock(or spin) - Busy Waiting
---------

*pthread_spin_xxx* 계열의 함수들을 통해 제공하는 락 방식으로 **Busy Waiting** 형태로 락을 구현합니다. 

동작 방식에 대한 이해를 돕기 위해 *pthread_spin_xxx* 함수를 간략하게 C 언어 형태로 표현한다면 아래와 같이 작성을 할 수 있습니다.

```cpp
int lock_val = 0; // 락 상태를 저장하기 위한 전역 변수 
...
int pthread_spin_lock (int* lock_ptr) {     // ref-A
    while (true) {                          // ref-C, Busy Waiting 
        if ((*locl_ptr) == 0) {
            (*lock_ptr) = 1;
            return 0;                       // ref-B
        }
    }
    return EDEADLK;
}
int pthread_spin_unlock(int* lock_ptr) {    // ref-A
    while(true) {                           // ref-C, Busy Waiting
        if ((*lock_ptr) == 1) {
            (*lock_ptr) = 0;
            return 0;                       // ref-B
        }
    }
    return -1;
}
...
void my_function() {
    ...
    pthread_spin_lock(&lock_val);   // lock_val 변수값을 0 -> 1로 변경 
    critical_section_code(); 
    pthread_spin_unlock(&lock_val); // lock_val 변수값을 1 -> 0로 변경 
}
```

> 위의 코드는 의사 코드로 실제 코드는 이보다 좀 더 복잡하며 CPU/OS에 따라 구현 방식에도 조금씩 차이가 있습니다

spinlock은 호출자가 락의 획득을 시도하는 시점 (상기 코드의 ref-A) 부터 락이 얻어지는 시점 (상기 코드의 ref-B) 때까지 SW 제어권이 호출자 (위의 예에서 my_function) 에게 돌아가지 않고, spinlock 함수내의 무한루프 (상기 코드의 ref-C)내에서 무한 대기를 하는 **Busy Waiting** 방식이기 때문에 **락 획득시까지 CPU 부하를 상승**시킨다는 특징이 있습니다. 

그러므로 spinlock을 사용하려고 하는 경우에는 상기 예제의 critical_section_code는 <u>**수행 시간이 상당히 짧고 수행 시간이 일정하며 (deterministic)  critical_section_code에서 blocking I/O 특성을 가지는 OS 호출등을 사용하지 않는 경우**</u>에는 제일 효율적인 방법이라고 할 수 있습니다.

spinlock의 잘못 사용한 예를 한번 보겠습니다.

> 조건 
> - 쓰레드 두 개 (A, B)를 구성
> - critical_section_code는 사용자 입력을 받고 저장

> 상황  
> - A 쓰레드가 락을 획득한 후 사용자 입력 대기를 위해 getchar() 호출
> - getchar()는 기본적으로 blocking I/O 이므로 A 쓰레드는 비활성 상태로 전환
> - OS 스케쥴러가 B 쓰레드를 활성 상태로 전환
> - B 쓰레드에서 락의 획득을 시도 
> - B 쓰레드는 무한루프 대기 수행 (A 쓰레드가 spinlock을 점유한 상태에서 Sleep 상태로 전환)


> 예상 결과
> - 사용자 입력이 발생해서 OS가 A 쓰레드를 활성 상태로 전환할 때까지 
> - B 쓰레드가 구동중인 CPU Core는 부하율 100%를 유지

spinlock은 위와 같은 문제를 야기할 수는 있어 사용 시 주의가 필요하긴 하지만 락을 획득하기 위한 내부 로직의 복잡도나 쓰레드간 절체를 위한 부하가 다른 방식에 비해 상대적으로 작기 때문에 (OS 개입없이 락의 획득이 이루어짐) 상당히 빈번하게 락을 수행해야 하는 경우에 주로 사용됩니다. 
예를 들면 쓰레드간에 데이타/이벤트등의 전달을 위해 사용하는 FIFO, LIFO, Queue 등의 구조에 대한 동시 접근 제어가 필요한 경우에 많이 사용됩니다.

----------

mutex - context switching if not available
---------

이제부터 설명하는 pthread 락 함수들은 처음에 설명한 spinlock과는 다르게 락의 획득과 반환 과정에 모두 OS가 개입을 하게 됩니다. 

일단 OS가 개입되면 다양한 방식으로 락을 획득할 수 있습니다. 예를 들어 획득까지의 대기 시간을 설정할 수도 있으며 대기중인 동안 CPU를 다른 쓰레드가 사용할 수 있도록 넘길 수도 있습니다. (spinlock은 이런 것들이 모두 불가능합니다)

이번에 설명할 mutex가 위와 같은 다양한 조건으로 락을 거는 것이 가능하며 pthread에서는 *pthread_mutex_xxx*  함수들을 통해 제공됩니다.
실제 mutex를 구현하는 내부 로직은 커널에 대한 이해를 포함하여 기술적으로 상당히 어려운 내용들이 많기 때문에 spinlock과의 차이를 확인하기 위한 정도로만 간략하게 C 언어로 동작 방식을 표현한다면 아래와 같이 작성을 할 수 있습니다.

```cpp
int lock_val = 0; // 락 상태를 저장하기 위한 전역 변수
...
int pthread_mutex_lock (int* lock_ptr) {
    if ((*lock_ptr) == 0) {
        (*lock_ptr) = 1;    
        return 0;
    }
    else {
        errno = add_to_fifo_and_wakeup_me_at_my_turn (); 
        // 상기 함수 호출 후 쓰레드는 비활성 상태로 전환됨

        // 아래 코드는 send_signal_to_wakeup_others에 의해 활성화되면 수행됨
        (*lock_ptr) = 1;    

        return 0;
    }
}
int pthread_mutex_unlock(int* lock_ptr) {
    if ((*lock_ptr) == 1) {
        (*lock_ptr) = 0;
        send_signal_to_wakeup_others(); // 이 함수 호출 후 비활성화되지 않음 
        return 0; 
    }
    errno = -1; 
    return 0; 
}
...
void my_function() {
    ...
    pthread_mutex_lock(&lock_val);   
    critical_section_code();
    pthread_mutex_unlock(&lock_val);
}

```
> 구체적으로 표현하기가 복잡한 부분은 아래와 같이 기능을 표현하기 위한 두개의 함수로 단순하게 표현해 봤습니다. 

> add_to_fifo_and_wakeup_me_at_my_turn :  OS가 나중에 쓰레드를 활성화할 수 있도록 mutex 구조체 내의 FIFO 대기열에 쓰레드 정보를 추가해서 OS로 전달하고 OS에게 다른 쓰레드/프로그램을 활성화하도록 요청합니다. (나 자신은 락을 대기하여야 하므로 비활성 상태로 전환됨)

> send_signal_to_wakeup_others :  대기중이던 쓰레드들이 활성화되어 락을 획득할 수 있도록 (mutex내의 FIFO 리스트에 등록된 순서대로) OS에게 이벤트를 발생시킵니다. 함수 호출 시 쓰레드는 비활성화가 되지  않습니다. OS는 비활성 상태인 대기 쓰레드를 다른 CPU/Core에서 동시 수행시킬 수 있다고 판단하면 대기중인 쓰레드를 바로 활성화 시킬 수 있습니다.

mutex는 spinlock의 Busy Waiting 과는 다르게 락을 획득할 수 없다면 SW 실행 권한을 OS로 넘기고 비활성 상태로 전환하기 때문에 **Busy Waiting이 발생하지 않는다**는 특징이 있습니다.

또한 spinlock의 경우에는 락 획득을 대기중이던 쓰레드들 중 어느 쓰레드가 락을 획득할 것인지 순서를 제어할 수 없지만 mutex를 사용하게 되면 mutex 내부의 FIFO 대기열에 락을 요청한 쓰레드들을 등록하기 때문에 락의 획득은 항상 요청을 한 순서대로 획득이 이루어지게 됩니다.
일종의 Fairness 형태를 제공하는 이러한 FIFO 대기열 관리는 일부  쓰레드들이 락을 독점해서 **critical section의 수행이 일부 쓰레드들로 편중되는 현상을 막을 수 있다**는 장점이 있습니다.

이 외에도 *pthread_mutex_timed_lock*을 이용하여 일정 시간내에 락을 확득할 수 없는 경우에는 락의 획득을 포기하도록 할 수 있고 (대기중에는 쓰레드는 비활성 상태를 유지하고 타임아웃 처리는 OS에서 수행됨) 락에 대한 **재귀 호출** (my_function에 대한 재귀 호출을 통해 동일한 락을 반복적으로 획득해야 하는 경우)도 가능합니다.  

mutex는 spinlock에서는 제공하지 못하는 **CPU 점유 시간의 최소화 / Fairness / 재귀호출 / 타임아웃**과 같은 장점이 있어서 범용적으로 제일 많이 사용하는 락 방법이지만, 이런 특징을 제공하기 위해서 spinlock에 비해 락의 획득을 위해 소요되는 오버헤드가 클 뿐만 아니라 쓰레드의 비활성화/활성화의 반복을 위한 OS의 context switching 부하와 타이머 관리를 위한 오버헤드가 추가되게 됩니다. 

프로그램내에서 spinlock과 mutex 중 어느 형태를 사용하는 것이 적합할 것인가는 **락의 횟수** 및 **critical section 의 수행 시간**, **락을 공유하는 쓰레드 수**등이 복합적으로 영향을 주기 때문에 초기 구현시에는 mutex를 이용하도록 구현을 한 후에 선택적으로 spinlock을 적용하는 것도 좋은 구현 순서라고 할 수 있습니다. 


----------

rwlock - conditional context switching 
---------

이번에 설명할 것은 rwlock으로 pthread에서는 *pthread_rwlock_xxx* 함수들을 통해 제공됩니다. 

기본적으로는  CPU 점유 시간의 최소화 / Fairness / 재귀호출 / 타임아웃과 같은 mutex의 특징을 동일하게 가지고 있습니다.

mutex에서는 critical_section_code의 자료/자원 접근이 Read 형태인지  Write 형태인지를 구분하지 않지만, rwlock에서는 **Read or Write**로 구분하여 락의 획득 시 비활성 상태로의 천이 (OS의 context switching 부하)를 최소화 할 수 있는 특징이 있습니다. 

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
    else {                      // if already locked 
        if ((*reader) > 0) {    // ref-A, if locked by reader 
            (*reader) ++;       // Just increase reader count 
        }
        else {                  // if locked by writer 
            errno = add_to_read_fifo_and_wakeup_me_at_my_turn ();
            // 상기 함수 호출 후 쓰레드는 비활성 상태로 전환됨

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
        errno = add_to_write_fifo_and_wakeup_me_at_my_turn (); 
        // 상기 함수 호출 후 쓰레드는 비활성 상태로 전환됨

        (*lock_ptr) = 1;
        return 0;
    }
}

int pthread_rwlock_unlock (int* lock_ptr, int* reader) {
    if ((*lock_ptr) == 1) {
        if ((*reader) > 0)  {   // if locked by reader
            (*reader)--; 
            return 0;
        }
    
        // if locked by writer or no more reader 
        (*lock_ptr) = 0;
        if (send_signal_to_wakeup_writer() == 0) {  // 1st : check writer FIFO 
            send_signal_to_wakeup_reader(); // 2nd : check reader FIFO
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
    critical_section_code(); 
    pthread_rwlock_unlock(&lock_val, &lock_reader);
}
```
> mutex와 비교할 수 있도록 가급적 mutex와 비슷한 형태를 가지도록 표현을 했습니다.  

> add_to_xxx_fifo_and_wakeup_me_at_my_turn :  앞서 설명했던 mutex의 add_to_fifo_and_wakeup_me_at_my_turn와 동일한 기능을 수행합니다. 단, rwlock의 내부에는 reader와 writer로 구분된 두가지의 FIFO가 있어 호출한 함수에 따라 쓰레드가 저장되는 FIFO의 종류가 달라집니다.

> send_signal_to_wakeup_wrter :  mutex와 유사하게 대기중이던 쓰레드들이 락을 획득할 수 있도록 OS에게 이벤트를 발생시킵니다. 단, reader FIFO와 writer FIFO 중 writer FIFO에 대기중인 쓰레드들만 확인합니다. 

> send_signal_to_wakeup_reader : reader FIFO와 writer FIFO 중 reader FIFO에 대기중인 쓰레드들만 확인합니다. 


rwlock의 특징은 FIFO 대기열이 reader FIFO와 writer FIFO로 구분된다는 것 외에 위의 pthread_rwlock_rdlock() 함수에 보이는 것처럼 **락이 걸려 있는 상태지만 reader가 락을 건 상태(ref-A)라면** 비활성 상태로 전환하지 않고 락을 획득한 것으로 처리한다는 점입니다. 

이런 조건을 추가되게 되면 pthread_rwlock_rdlock()을 호출하는 하나 이상의 쓰레드는 동시에 critical section내의 코드를 수행하는 게 가능해 지므로 모든 쓰레드를 직렬화하는 mutex에 비해 쓰레드들의 동시 실행성과 Cntext Switching으로 인한 성능 저하가 모두 개선될 수 있습니다. 

물론 다수의 쓰레드가 critical_section_code내의 자원을 안전하게 동시 접근을 하기 위한 방법으로 자원을 변경(write) 하지 않고 참조(read)만 하는 방법이 일반적으로 많이 쓰이기 때문에 함수의 이름 역시 rdlock(read-lock)이라고 붙여지게 된 것이고 이러한 reader/writer의 특징을 부여하는 것은 critical_section_code내에서 프로그래머가 구현하여야 합니다. 

이런 특징을 잘 활용할 수 있는 코드의 예라면 데이타베이스와 같이 자료를 생성하는 생성자가 하나 또는 수개 이하로 한정되어 있고, 자원을 참조하는 소비자는 불특정의 다수인 경우라고 할 수 있습니다.

----------

semaphore 이건 설명하지 말자. 
---------

이번에 설명할 것은 pthread 라이브러리내의 락 함수는 아니지만 멀티쓰레드에서 락을 위한 
rwlock으로 pthread에서는 *pthread_rwlock_xxx* 함수들을 통해 제공됩니다. 


