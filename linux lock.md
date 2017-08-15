멀티쓰레드 락 (Lock)
============

이번에는 리눅스 멀티쓰레드 환경에서 빈번하게 사용되는 pthread 계열 락의 종류 및 각각의 특징과  프로그래머가 상황에 따라 간단하게 아토믹 함수를 이용하여 락을 만드는 방법에 대해서 얘기하고자 합니다.

 락의  주요 사용 목적은 다수의 쓰레드가 특정 자료 영역에 대한 (Read Only 접근을 제외한) **Read & Write** 또는 **Write & Write** 형태의 동시 접근을 차단하여 자료에 **안전**하게 접근할 수 있도록 하는 것이라고 할 수 있고 어떤 방식의 락을 사용해도 이러한 목적은 달성할 수가 있습니다. 

하지만 특정 자료 접근을 다수의 쓰레드들이  <u>**순차적**</u>으로  접근하도록 만드는 것이기 때문에 필연적으로 멀티쓰레드 프로그램의 동시 실행성의 저하 (성능 저하) 를 발생시킬 수 밖에 없으므로, 각각의 락 방식에 대한 이해를 통해 **동시 실행성의 저하를 최대한 억제**할 수 있는 방식을 사용할 필요가 있습니다.  

여기서는 각각의 락에 대한 기본적인 사용 방법이 아닌 락이 동작하는 방식에 대한 이해를 돕는 얘기를 하도록 하겠습니다. 

-------

spinlock - busy loop (pthread_spin_ ...)
---------

Description 

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

