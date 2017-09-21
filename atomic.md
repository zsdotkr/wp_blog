ATOMIC 연산 (Atomic Operation)
============

이번에 다루는 내용은 Atomic 연산(Atomic Operation)에 대한 내용입니다. [aaa][id_aaa]

SW를 싱글 쓰레드로만 구현한 경우에는 신경을 쓸 필요가 없지만, 멀티쓰레드를 사용하는 경우에는 락 (Mutex, SpinLock 등) 만큼 중요한 것이 이번에 설명할 Atomic 연산이라고 할 수 있습니다.

일단 왜 Atomic 연산에 대한 고려가 멀티쓰레드에서 중요한 것인지를 이해하기 위해서는 CPU에서 연산을 위해 데이타가 어떻게 관리되는지를 먼저 이해할 필요가 있습니다.

[id_aaa]:   this is ?? 

## CPU의 데이타 저장소 (레지스터 / 캐시 / 메모리) 
--------

우리가 작성한 SW의 실행 이미지는 일반적으로 영구 저장 장치 (HDD, Flash 등)에 저장되어 있으며, SW의 수행이 필요한 시점에 주 메모리 (DDR, SDRAM, SRAM 등)에 로딩되며 SW내에서 연산/생성/참조되는 데이타들 역시 기본적으로는 모두 주 메모리안에 저장된다는 된다는 것은 모두 알고 계실 겁니다.

어셈블러를 사용하거나 특수한 경우 (Intel DPDK와 같은 고속 패킷 핸들링)를 제외하고 일반적인 SW에서는 주 메모리 (또는 디스크등) 만 보이지만 CPU와 주 메모리 사이에는 아래와 같이 추가적인 데이타 공간이 존재하고 있다는 것도 어느정도 아실 것이라고 생각합니다. 

```
+-------------------------------------------------------+
|                       CPU Chip                        | 
|  +-----------------------------------+  +------+      |
|  |               Core                |  | Core |      |
|  | +--------------+ +--------------+ |  |      |      |
|  | | Hyper Thread | | Hyper Thread | |  |      |      |
|  | | +----------+ | | +----------+ | |  |      |      |
|  | | | Register | | | | Register | | |  |      |      |
|  | | +----------+ | | +----------+ | |  |      |      |
|  | +-------+------+ +------+-------+ |  |      | .... |
|  |         |               |         |  |      |      |
|  |    +----+---------------+----+    |  |      |      |
|  |    |        L1 Cache         |    |  |      |      |
|  |    +------------+------------+    |  |      |      |
|  |                 |                 |  |      |      |
|  +-----------------+-----------------+  +--+---+      |
|                    |                       |          |
|  +-----------------+-----------------+  +--+---+      |
|  |             L2 Cache              |  |  L2  |      |
|  +-----------------+-----------------+  +--+---+      |
|                    |                       |          |
|  +-----------------+-----------------------+--------+ |
|  |                    L3 Cahcel                     | |
|  +--------------------------------------------------+ |
+-------------------------------------------------------+
```

> 레지스터 (Register) 
> - CPU 내부에 있는 특수 목적용의 저장 공간으로 모든 CPU가 가지고 있습니다.
> - 레지스터는 CPU가 처리하는 모든 종류의 연산에 사용되는 데이타 값을 임시 보관하는 Data Register류, 데이타가 위치한 메모리 주소를 저장하는 Address Register류, 수학적인 연산의 수행과 결과를 보관하기 위한 Arithmetic Register류와 특수 목적용 Register류 등으로 구분되며 수백 바이트 용량 (수십개의 레지스터)을 일반적으로 가지고 있습니다. 
 
> L1/L2/L3 Cache
> - CPU 내부/외부에 있는 저장 공간으로 거의 모든 CPU가 최소 하나 이상의 Cache 계층을 가지고 있습니다.
> - Cache는 CPU와 주 메모리 간의 데이타 Read/Write 에 대한 임시 저장 공간(캐시) 역할을 수행합니다. 
> - 상기 캐시의 구조는 일반적인 구성을 보여주는 것이고 실제적인 물리적 구성은 CPU별로 다릅니다. 

> Read/Write 속도 
> - Register Read/Write 속도 = CPU의 논리적인 최고 성능 
> - Register > L1 Cache > L2 Cache > L3 Cache > 주 메모리 


## 사칙 연산이 CPU에서 처리되는 방법 
-----

일단 대략적인 CPU의 구조를 살펴봤으니 이제 기본적인 사칙 연산을 수행하는 코드가 실제 CPU에서 어떻게 처리되는 지 알아보도록 하겠습니다. 

```cpp
int value;      // 전역 변수
int my_func() {
    value += 1;
    return value;
}

```
위의 함수를 컴파일한 후 CPU에서 실제 수행하는 어셈블러 코드로 역변환(Disassembly)을 하면 아래와 같은 어셈블러 코드가 생성됩니다.


```cpp
int my_func()
{
  push   rbp
  mov    rbp,rsp
    value += 1;
  mov    eax,DWORD PTR [rip+0x200b54]   // #1, value를 레지스터로 복사 
  add    eax,0x1                        // #2, eax 레지스터의 값을 증가
  mov    DWORD PTR [rip+0x200b4b],eax   // #3, 레지스터 값 (연산 결과)을 메모리로 복사
  
    return value;
  mov    eax,DWORD PTR [rip+0x200b45]
}
  pop    rbp
  ret

```

위의 코드는 인텔 x86용으로 컴파일한 바이너리를 objdump로 디어셈블 한 것으로 C 코드로 표현된 부분의 아래쪽에 위치한 어셈블러 코드들이 C 코드를 위해 만들어진 어셈블러 코드라고 보면 되고, *"value += 1"* 을 수행하기 위해 *"#1 ~ #3*" 까지의 세개의 어셈블리 코드가 수행된다고 이해를 하시면 됩니다.

코드 옆에 주석을 달아놓은 것처럼 하나의 변수를 중가하기 위해 CPU는 *"메모리내 데이타를 Register로 복사" -> "Register만을 이용한 데이타의 덧셈" --> "연산 결과를 가진 Register 값을 메모리로 복사"* 하는 **Read-Modify-Write** 라고 일반적으로 부르는 3단계 과정을 수행합니다.

## 조건문이 CPU에서 처리되는 방법 
-----

이번에는 *"if (비교)"* 에를 한번 보도록 하겠습니다.

```cpp
void my_func()
{
  push   rbp
  mov    rbp,rsp
    if (value == 0) {   value = 1;  }
  mov    eax,DWORD PTR [rip+0x200b54] // #1, value를 Register로 복사 
  test   eax,eax                      // #2, value - value 연산 수행 
  jne    4004f0 <my_func+0x1a>        // #3, "연산결과 != 0"이면 #6로 점프  
  mov    DWORD PTR [rip+0x200b46],0x1 // #4, value에 1을 저장 
    else            {   value = 0;  }
  jmp    4004fa <my_func+0x24>        // #5, #7으로 점프 
  mov    DWORD PTR [rip+0x200b3a],0x0 // #6, value에 0을 저장 
  pop    rbp                          // #7
  ret
```

위의 코드는 value란 전역변수를 0과 1 사이에서 토글시키는 간단한 함수입니다. 
이 예제 역시 *"if (value == 0)"*이란 판단을 하기 위해 *"#1 ~ #2"* 까지의 2개의 어셈블리 코드가 수행되고, *value* 란 값을 변경하는 데 다시 *"#3 ~ #6"* 까지의 다단계 절차를 수행함을 볼 수 있습니다. 

코드 옆에 주석을 달아놓은 것처럼 하나의 변수값을 비교하는 과정도 첫번째 예와 유사하게 CPU 입장에서는 뺄셈을 하는 것과 다를 게 없이 
*"메모리내 데이타를 Register로 복사" -> "Register만을 이용한 데이타의 뺄셈"* 을 수행한 뒤 *"연산 결과를 가진 Register 값의 결과를 비교" --> "메모리내의 데이타를 다른 값으로 변경 *" 하는 다단계의 과정을 거치게 되고 일반적으로 이런 비교 후 변경을 하는 과정은 **"Compare-and-Exchange"** 라고 부릅니다. 

> CPU에서 값의 비교란 *"메모리 데이타 == 상수"* 와 같은 직접적인 비교는 할 수 없고 *"사칙 연산의 수행 결과 == zero or plus or minus"* 만을 판단할 수 있기 때문에 위와 같은 수행 절차를 거치게 됩니다.

당연할 걸 수도 있지만 CPU가 모든 연산을 메모리를 직접 사용하지 않고 Register란 중간 저장소를 이용하는 이유는 아래와 같습니다. 

* Register는 메모리 또는 캐시에 비해 CPU의 제어 로직이 접근하는 속도가 월등히 빠릅니다. (Register의 접근 속도 = CPU의 성능)
* 하나의 연산 결과는 연이은 또다른 연산에 재사용될 가능성이 상당히 높기 때문에 연산 결과를 Register 내에 보관하고 있는 것이 성능 개선에 유리합니다.
* 메모리는 단순한 저장장치로 연산을 수행할 수도 없고, 특정 데이타를 다른 영역으로 직접 복사/이동시킬 수도 없습니다.
* 만약, 연산 메모리가 연산 장치를 가진다면 CPU라고 부르는 게 맞고 실제 최근의 CPU들은 내부 공간의 절반 정도는 메모리 (Cache)를 만들기 위한 공간으로 사용합니다.


## ATOMIC 연산이란
-----

자 이제 기본적인 배경 설명을 마쳤으니 Atomic 연산이 무었인지와 왜 필요한 것인지를 보도록 하겠습니다. 

위에서 첫번째로 예를 든 Read-Modify-Write 형태의 연산이나 두번째 예인 Compare-and-Exchange가 멀티쓰레드에서 수행된다면  

* *value* 가 전역변수이므로 모든 쓰레드가 접근을 할 수 있고
* CPU 역시 다수의 Core를 가지고 있고 Core별로 각각의 독립적인 Register를 가지고 있기 때문에 (Intel의 경우는 HT별로 1벌씩)

첫번째 예에서 만약 4개의 Core가 동시에 *"value = 0"* 에 대해 *"value += 1"* 절차를 수행한다면 4번의 연산이 수행되므로 *"value +=4"* 가 되어야 하겠지만 *"#1 ~ #3*" 과정이 다수의 Core에서 독립적인 Register내에서 동시에 수행되고, *#2* 에서 Register는 모두 0이라는 값을 가지고 덧셈을 수행하므로 결론적으로 *"value += 1"* 인 결과가 나타납니다.물론 위의 상황 이외의 A라는 쓰레드가 #2를 실행중인데 B 라는 쓰레드가 #1을 시작하는 경우, A는 #3를 수행하는 데 B는 #1 또는 #2를 수행하는 경우 등 더 많은 상황이 있을 수 있습니다. 

두번째 예에서는 *"#1 ~ #3"* 까지만 수행한다면 다수의 Core가 동시에 수행해도 문제가 없지만 (메모리에 대한 Read Only 참조) 곧이어 참조한 변수를 바꾸는 *"#4, #6"* 과정이 있기 때문에 다수의 Core가 (거의) 동시에 수행을 하게 되면 *"#1 ~ #3"*의 수행과 *"#4, #6"*의 수행이 ㅓㄲ이게 되면서 최종적으로 "value"의 값은 SW에서 의도하지 않는 형태가 될 가능성이 높아집니다. 

그러므로 멀티쓰레드 환경에서는 이러한 다단계로 수행되는 연산을 안전하게 할 수 있는 방법이 필요하게 되었고 멀티코어 CPU가 **어셈블러 차원에서 다단계의 명령을 하나의 트랜잭션으로 처리할 수 있도록 지원하는 하나 이상의 어셈블러 명령 조합**들을 **Atomic 연산**이라고 합니다.  

## ATOMIC 연산의 예
-----

기본적으로 Atomic 연산은 위에서 설명한 **Read-Modify-Write** 및 **Compare-and-Exchange** 를 하나의 트랜잭션으로 처리할 수 있도록 제공됩니다. 

GCC를 기준으로 하면 GCC built-in 함수 중 *"__sync_..."* 프리픽스를 통해 제공되는 함수들이 atomic 연산용 함수라고 볼 수 있습니다. 

> 여기서 말하는 GCC는 GCC v4.4 또는 그 이전부터 아직까지는 지원중인 __sync 계열 built-in을 기준으로 설명합니다. 
> 아직도 GCC 4.9 이전 GCC를 사용할 수 밖에 없는 환경 (ARM, MIPS등)이 많이 있는 관계로 
> GCC v4.8 (현실적으로는 GCC v4.9) 이상에서 지원하는 C11 표준상의 atomic을 위한 __atomic 계열 built-in에 대한 내용은 
> 기반 지식은 여기서 설명하고자 하는 내용과 동일하므로 나중에 다루도록 하겠습니다.

일단 제공되는 함수들은 아래와 가은 종류들이 있으며, 여기서는 Atomic 연산을 이해하는 것이 목적이므로 "Case-A" 및 "Case-B" 함수들의 특징을 살펴보는 것으로 하고 Memory Barrier 에 대한 설명이 필요한 "Case-C" 부분은 따로 정리를 하도록 하겠습니다.

> Case-A) Read-Modify-Write를 위한 ATOMIC 연산 함수 
> * 변수에 대한 연산을 수행하고 연산 수행 전의 값을 반환 (__sync_fetch_and_add 등)
> * 변수에 대한 연산을 수행하고 연산 수행 후의 값을 반환 (__sync_add_and_fetch 등)
> * 지원되는 연산의 종류 : +, -, AND, OR, XOR, NAND
> * 사용 가능한 변수의 종류는 1, 2, 4, 8 바이트 정수형만 가능

> Case-B) Compare-and-Exchange를 위한 ATOMIC 연산 함수 
> * 변수가 A와 같으면 B라는 값으로 변경 (__sync_bool_compare_and_swap 등 )
> * 사용 가능한 변수의 종류는 1, 2, 4, 8 바이트 정수형만 가능

> Case-C) 기타
> * __sync_synchronized
> * __sync_lock_test_and_set 
> * __sync_lock_release 

일단은 Read-Modify-Write를 위한 Atomic 함수를 보도록 하겠습니다.
아래 코드는 처음에 사용했던 사칙 연산의 코드 아래에 __sync_fetch_and_add 란 Atomic 덧셈 함수를 추가한 후 디어셈블한 코드입니다. 

```cpp
int value; 
int my_func()
{
  push   rbp
  mov    rbp,rsp
    value += 1;
  mov    eax,DWORD PTR [rip+0x200b54]   // #1, value를 레지스터로 복사
  add    eax,0x1                        // #2, eax 레지스터의 값을 증가
  mov    DWORD PTR [rip+0x200b4b],eax   // #3, 레지스터 값 (연산 결과)을 메모리로 복사

    __sync_fetch_and_add(&value, 1);    // GCC의 Atomic 덧셈 함수 
  lock add DWORD PTR [rip+0x200b43],0x1 // #4, value 주소값이 있는 데이타를 1만큼 증가
    return value;
  mov    eax,DWORD PTR [rip+0x200b3d]
}
  pop    rbp
  ret
```

위의 코드를 보면 __sync_fetch_and_add를 사용하면 *"#4"* 처럼 한줄의 어셈블 명령으로 *"value +=1"* 을 수행하는 코드를 만들 수 있습니다.

"#4" 라인을 보면 "#2"처럼 Register를 이용해서 덧셈 연산을 하지 않고

1. *"value"* 에 해당하는 메모리 주소에 직접 덧셈을 하는 것을 볼 수 있고 
1. *"lock"* 이란 프리픽스도 사용된 것을 알 수 있습니다.

앞에서 이미 설명을 했지만 메모리는 단순한 저장장치이므로 메모리가 직접 덧셈을 수행할 수 없는데도 불구하고 "#4"의 코드는 메모리를 대상으로 뎃셈을 수행하는 것을 볼 수 있는데, #4의 어셈블러 코드는 실제로는 Atomic 연산을 위해 만들어진 특수한 형태의 어셈블러 명령으로 

* #1 ~ #3과 동일하게 Read-Modify-Write 절차를 수행하되 사용하는 레지스터가 보이지는 않습니다. (일종의 임시용 Register를 사용)  
* *"lock"* 이란 특수한 프리픽스는 Read-Modify-Write 수행 시 해당 주소를 다른 Core가 접근하지 못하도록 합니다
* *"lock"* 프리픽스의 뒤이어 나타나는 어셈블러 명령 1개가 수행되면 lock이 해제되는
* 일종의 CPU내에 하드웨어적으로 만들어 놓은 spinlock 이라고 볼 수 있습니다.

이번에는 Compare-and-Change를 위한 Atomic 함수를 보도록 하겠습니다.
아래 코드는 두번째 사용했던 비교 연산의 코드 아래에 __sync_bool_comapre_and_swap이란 Atomic 비교 함수를 추가한 후 디어셈블한 코드입니다. 

```cpp
void my_func()
{
  push   rbp
  mov    rbp,rsp
    if (value == 0) {   value = 1;  }
  mov    eax,DWORD PTR [rip+0x200b54] // #1, value를 Register로 복사 
  test   eax,eax                      // #2, value - value 연산 수행 
  jne    4004f0 <my_func+0x1a>        // #3, "연산결과 != 0"이면 #6로 점프  
  mov    DWORD PTR [rip+0x200b46],0x1 // #4, value에 1을 저장 
    else            {   value = 0;  }
  jmp    4004fa <my_func+0x24>        // #5, #7으로 점프 
  mov    DWORD PTR [rip+0x200b3a],0x0 // #6, value에 0을 저장 

  __sync_bool_compare_and_swap(&value, 0, 1); // GCC의 Atomic 비교 함수 
  mov    eax,0x0                      // 비교할 값 (0) 을 eax 레지스터에 저장 
  mov    edx,0x1                      // 변경하려는 값 (1)을 edx 레지스터에 저장 
  lock cmpxchg DWORD PTR [rip+0x200b28],edx // #7, if (value == 0) { value = 1; }
  nop
  pop    rbp
  ret
```

위의 코드를 보면 __sync_bool_compare_and_swap을 사용하여 value의 값이 "0"이면 "1"로 변경하는 코드를 만들 수 있고 #1 ~ #4까지의 코드를 #7의 한줄로 대체할 수 있음을 볼 수 있습니다. 

"#7"의 코드 역시 첫번째 예처럼 메모리를 대상을 직접 할 수 없는 비교 연산 (뺄셈)을 "Read-Modify-Write"를 임시 레지스터를 이용하여 수행한 것이고, Lock 프리픽스를 이용해서 해당 주소에 대한 "Read" ~ "Write" 동안 해당 주소를 다른 Core가 접근하지 못하도록 Lock을 건 것이라고 이해를 하시면 됩니다. 

단, *"else { value = 0; }"* 에 해당하는 "#6 ~ #7" 부분은 Atomic으로는 처리할 수가 없습니다. 

왜냐하면 __sync_bool_compare_and_swap이 지원하는 범위는 "한번의 비교와 변경"만 가능하고 "else"와 같은 에외 처리를 지원하지 않으며, __sync_bool_compare_and_swap 수행 직후에 이미 "value"에 해당하는 주소는 어느 core도 접근이 가능해졌고 누군가 변경했을 가능성이 있기 때문에 "else {value = 0; }"을 추가한다고 해도 atomic을 보장할 수 없기 때문입니다.  

## ATOMIC 연산의 한계

Atomic 연산이 할 수 있는 범위는 딱 여기까지라고 보시면 됩니다. 

* Atmic 연산은 정수형에만 사용이 가능하고 실수형은 사용할 수 없습니다. 
* Atomic Comapre-and-Swap은 "비교 후 변경" 1회가 가능합니다. 

두번째의 예에서 Atomic 연산을 통해

*else*에 해당하는 "if (vlaue != 0) { value = 0; }" 는 __sync_bool_compare_and_swap을 한번 더 써서 처리를 할 수는 있겠지만 이는 CPU가 제공하는 lock 프리픽스를 두번 써야만 하는 것이고  Atomic 범위  

```cpp

```

면 처리할 수는 있지만 lock 프리픽스의 유효 점우는 1줄의 어셈블러 명령이므로 두번째 


* 두번째의 예에서 dnjsfo zhemdp dlTsms 
* 

dml zhe durtl 
불행히도 원본의  



쓰다가 만 것 
----

Atomic 연산이 
* x86의 경우 1개의 메모리 영역에 대해 연산 시 CPU의 메모리 억세스 전체를 중지시키는 형태로 제공 (LOCK  Prefix ) 

이라고 할 수 있고 GCC built-in 에서는 Atomic 연산의 대상 데이타의 크기 (1, 2, 4, 8 바이트등)에 대한 제한을 두고 있진 않지만 실제로 다양한 CPU에 모두 적용을 고려해야 한다면 소프트웨어에서 사용하는 Atomic 변수의 크기는 4 바이트 (int, unsigned int) 또는 8 바이트 (int64_t, uint64_t, CPU가 64비트인 경우) 정도로만 사용할 필요가 있습니다.



x86의 경우에는 1 ~ 8 바이트 정수형에 대해 모두 Atomic 연산 기능을 제공하지만 CPU 마다 Atomic 연산 기능의 구현 방식이 틀리고 제한적이기 때문에 
모든 CPU 종류에 대해서 GCC가 built-in을 제공하지도 않기 때문에  


참고 
---------

ATOMIC 연산을 사용하지 않는 경우에 대한 연산 오류 예 

> 조건 
> - 쓰레드 2개를 구동하고 하나의 전역 변수에 대해서  
> - A 쓰레드는 *"value += 1"* 을 2억회 수행
> - B 쓰레드는 *"value -= 1"* 을 1억회 수행
> - 쓰레드가 모두 종료되면 전역 변수의 최종값을 출력 

> | 회차 |   1 | 2 | 3 | 4 | 5 | 
> | --- | --: | --: | --: | --: | --: | 
> | 결과 | 부정확 | 부정확 | 부정확 | 정확 | 정확 | 
> | 최종값 | 100477358 | 97968720 | 97968720 | 100000000 | 100000000





상기 예에 언급한 어셈블러 코드를 생성하는 방법 

> C 소스 컴파일하는 방법 : gcc -g -o sample sample.c 
> - -g : 실행파일내에 디버깅 정보를 추가합니다. 이 정보를 이용하여  Disassembly 시 어셈블리 코드내에 C 코드 참조 위치를 표시할 수 있습니다. 
> - -o : 컴파일된 실행 파일명을 지정합니다. 위 예에서는 *sample*이 실행파일입니다.
> - -Ox 계열의 최적화 옵션을 주게 되면 최적화로 인해 C 코드 참조 위치가 부정확하게 나옵니다. 
 

> Disassembly 생성 방법 : objdump -S -d -M intel-Mnemonic out 
> - -S : 생성된 어셈블러 코드내에 참조용 원본 C 코드를 추가합니다. 단 확장자를 제외한 실행파일명과 소스 파일명이 같아야 합니다.  
> - -d :objdump 프로그램에게 Disassembly 를 요청합니다.  
> - -M intel-Mnemonic : 어셈블러 코드를 인텔 어셈블러 형태로 출력합니다. 


Links 
---------

* https://en.wikipedia.org/wiki/CPU_cache
* https://en.wikipedia.org/wiki/Scratchpad_memory
* https://www.nextplatform.com/2016/05/31/intel-lines-thunderx-arms-xeons/

* http://demin.ws/blog/english/2012/05/05/atomic-spinlock-mutex/
