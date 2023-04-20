Kernel-Module > Kernel안에 module 형태로 삽입
라즈베리파이가 어차피 리눅스도 설치가 되고 리눅스가 있으면 gcc도 사용할 수 있으니까 필요한 것들을 라즈베리파이에서 구현할 수 있지 않을까 하지만 라즈베리파이는 arm 기반의 저전력 프로세서가 있기 때문에 컴파일하려 하면 시간이 많이 걸리고 모니터 등을 연결하는 것이 쉽지 않다. 그래서 소프트웨어를 개발하고 컴파일 하는 것을 노트북에서 한다. 
그런데 pc 대부분은 arm이 아니라 intel등의 cpu를 사용한다. cpu종류가 다르면 그 cpu가 이해하는 머신 instruction도 다르게 되는데 즉, 컴파일시 생성되는 바이너리가 intel cpu를 위한 것과 arm cpu를 위한 것이 다르다. 
intel cpu를 사용하는 데서 소프트웨어를 개발하고 컴파일했을 때, 그것을 그대로 arm기반의 라즈베이파이에서 설치 또는 실행하는데 인식이 제대로 되지 않는다.

> 그래서 intel 기반의 컴퓨터에서 소프트웨어를 개발했으나 *컴파일 자체는 arm cpu가 이해하는 기계어 코드로 컴파일 하는 것*ㄴ을 크로스 컴파일이라고 한다. 즉, 타겟이 되는 시스템이 이해하는 바이너리를 생성하는 것을 크로스 컴파일이라고 한다.
# 커널 모듈 컴파일시에 크로스 컴파일러가 필요한 이유

cpu 종류가 다르면 컴파일시 만들어지는 바이너리 코드가 달라서 서로 이해를 할 수 없는데
arm 기반의 cpu가 사용되는 라즈베리파이에서 실행될 소프트웨어를 arm이 아닌 intel 기반의 cpu가 사용되는 다른 컴퓨터에서 만들기 때문에, 타겟이 되는 시스템이 이해하는 바이너리를 생성하는 크로스 컴파일을 해야한다. 


# Background

## Operating System vs Kernel

### Linux Distro(OS)
> 리눅스 커널에 기반을 둔 software collection 으로 만들어진 운영 체제

* OS안에 커널이 포함된다.

* cpu virtualization, memory virtualization, lock을 위한 primitive들, storage를 위한 file system이 kernel안에 구현되어 있다.   

( 일반적으로 device driver는 kernel안에 포함되었다고는 말하지 않는다.   하지만 kernel을 배포할 때 대표적인 device driver들은 같이 포함되어 배포된다. )

## Types of Kernel
> kernel은 구현되는 형식에 따라서 구분된다. 일반적으로 경험하는 os의 커널들은 Monolithic kernel의 형태로 되어 있다.

### Monolithic kernel
* 하나의 아주 큰 binary 이미지로 된 kernel이다. (하나의 이미지)

* 일반적으로 kernel은 여러 개의 파일로 구성되어 있지만 kernel을 컴파일하고 나면 os kernel image가 하나의 큰 binary file로 생성된다

* cpu scheduling, memory virtualization, file system에 대한 기능이 하나의 큰 image(하나의 address space)안에 포함되어 있기 때문에 기능들을 구현하기 위한 자료구조, 함수(routine)에 접근하기 쉽다. 따라서 성능이 좋다. (운영체제가 실행하는데 있어 overhead가 상대적으로 적다.)   

* 반면에 하나의 큰 binary file로 되어 있기 때문에 수정하기가 어렵다.
(일부분을 수정하더라도 kernel 전체를 compile해야 하기 때문에 다시 새로운 하나의 image file 생성)

* 또 일부분을 고쳤을 때 다른 쪽에 dependency 발생할 수 있다.


### Microkernel
* 커널의 기능들을 조각내서 별도의 unit으로 관리한다. 

* 즉, 각각의 기능들은 별도의 binary image를 생성한다.(각각 서로 다른 address space를 갖고 있다.)

* 기능들 간에 dependency를 없애줬기 때문에 통신(IPC)을 통해 요청을 하고 받는다.

* 기능들을 모두 다른 unit으로 만들어 놓고 가장 기본적으로 제공되어야 하는 기능을 모아놓은 부분을 MicroKernel이라고 한다. 

* cpu 관리, memory 관리등 이런 기능은 최대한 kernel 밖으로 빼서 유닛으로 두고 kernel 안에서는 그 유닛들 간의 통신과 같은 기본적인 기능만 포함한 작은 커널(microkernel)을 둔다.

* microkernel에서 제공하는 특정 unit을 고쳐도 통신 프로토콜만 지키면 기능이 수정되어도 다른 쪽에 영향을 주는 dependency가 최소화되기 때문에 유연하다.

* 시스템을 network link(TCP/IP)로 분산되게 할 수 있지만 성능이 낮아지기 때문에 일반적으로 분산된 경우는 별로 없다.

* 분산 노드를 적용하지 않아도 IPC와 같은 통신 overhead 때문에 퍼포먼스가 낮다.

## Linux Device Drivers
* Linux Kernel과 같이 배포되는 device driver들의 수가 많고 크기가 크기 때문에 device driver 모두를 Kernel 이미지 안에 넣게 되면 Kernel이 너무 커진다. 이런 문제를 해결하기 위해서 나온 것이 Kernel Module이다.
ㄴ
## Kernel Module
* MicroKernel의 장점을 Device Driver에 적용하기 위해서 만든 것이다.

* Kernel 이미지에 포함되지 않고 별도 이미지로 생성될 수 있기 때문에 Kernel 이미지를 작게 만들어 준다.

* 소스코드와 같이 배포되는 device driver 모두를 사용하진 않기 때문에(연결된 device에 해당하는 device driver만 사용) 쓸데없는 device driver까지 load하는 문제를 해결할 수 있다.

* 어떤 device driver를 수정했을 때 모든 시스템의 image를 새로 만들거나 그것을 적용하기 위해 시스템을 다시 부팅하는 것을 없도록 해준다.

* 커널 모듈은 시스템이 실행될 때 모듈 단위로 제거 혹은 load할 수 있다. 

## Kernel Module Programming(Loadable Kernel Modules)
* device driver 모델 마다 제공되어야 하는 함수들의 prototype(framework)이 정의되어 있다. 그래서 device driver에서 함수들의 body를 구현하면 framework의 틀과 device driver안에 실제 구현되어있는 body부분이 연결된다.

* standalone process가 아니라 함수들이 정의되어 있어서 운영체제(커널)가 필요할 때 호출한다.

* kernel module은 kernel space에서 동작한다.