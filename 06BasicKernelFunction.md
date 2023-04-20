## User Memory Access
### copy_to_user
> Kernel 영역에 있는 data를 user 영역에 있는 버퍼로 copy

* user영역에서는 응용프로그램이 실행되고 있는 응용 코드 안에서 kernel영역에 해당하는 주소번지를 pointer 변수에 넣고 억지로 접근한다면 error message가 발생하고 응용프로그램이 종료된다.

* (kernel 모듈이 실행되는)kernel영역 안에서 user영역의 주소를 마음대로 접근할 수 있다. 왜냐하면 kernel 모드로 cpu가 동작하기 때문에 user 영역에 있는 virtual address를 갖고 user의 data를 가져오든지, user 영역에  임의의 위치에 원하는 data를 넣을 수 있다.
  
* 그래서 이렇게 user 영역에 있는 주소번지를 표시했을 때 문제가 발생하진 않는다.

### copy_from_user 

## Memory Allocation(Kernel안에 메모리를 할당하는 방법)

### vmalloc
* 가상 메모리상에서 연속된 메모리를 할당한다.

> 가상 메모리에서 연속되었다는 것은 물리 메모리에서는 비연속적일 수 있다는 것이다. 
가상 메모리는 페이지 단위로 mapping되기 때문에 page 단위로 불연속할 수 있다. (user 영역에서 malloc으로 할당할 때도 마찬가지)

* 그러므로 DMA에는 적합하지 않다.

> DMA(Direct Access Memory)는 I/O를 실행하는 주변장치(Network card, Storage)들이 data를 메인 메모리에 집어 넣든지 메인 메모리로 부터 I/O장치로 data를 가져가기 위해서 사용하는 방법이다.
주변장치에게 알려줄때 물리주소의 시작주소와 길이만을 알려주기 때문에 vmalloc에 의해 할당된 메모리 영역에 대해서는 시작주소와 길이만 주면 실제 물리 메모리에서는 비연속적일 수 있기 때문에 다른 메모리 영역에서 data를 가져갈 수 도 있다.

* 가상환경에서 크고 연속적인 메모리 버퍼를 할당할 때는 적합하다. 왜냐하면 불연속적인 page를 가져다가 가상적으로 연속되어있는 큰 버퍼를 만들어 줄 수 있기 때문이다.

### kmalloc
* 물리적으로 연속된 메모리에 할당하는 방식이다.

#### GFP_KEREL
* 메모리를 할당할 때 내가 원하는 만큼의 사이즈의 버퍼 공간이 없는 경우 운영체제시간에 배운 swap, page eviction을 해줘야 하는데 이런 것들이 발생하게 되면 process는 재워야 한다.

#### GFP_ATOMIC
* 메모리를 할당하는데 실패하는 경우 잠들지 않고 해결할 때까지 계속버틴다.

* interrupt handler인 경우 이런 flag가 필요하다. interrupt handler는 process가 아니기 때문에 잠드는 scheduling을 적용할 수 없다.

#### GFP_DMA (참고만 하기)
* kmalloc을 사용하는 경우 page size보다 큰 버퍼에 대해서 물리적으로 연속된 페이지가 할당되는 것을 보장해준다.

* 물리주소인 경우 한정된 비트수로 표현되는 주소 공간에 할당되어야 DMA가 되는 조건이 있다. 이런 영역에 메모리할당이 필요한 경우에 사용한다.


### Slab Allocator
* Kernel안에서는 동일한 type(ex: 구조체 )이 반복적으로 할당되고 free되는 경우가 많다. 이런 구조체를 일종의 object로 보고 alloc, free를 효율적으로하기 위해 Slab Allocator가 구현되었다.

* 예를 들어, 일반적으로 현재 시스템들은 4kbyte짜리 page를 사용하는데 
4096byte의 page가 있다고 할때, 자주 쓰는 object의 size가 512byte라고 하면 페이지 하나를 미리 object가 8개 할당될 수 있도록 쪼갠다.
할당을 해달라하면 쪼개진 page 조각을 하나씩 준다.
free하라고 하면 진짜로 free하는게 아니라 "free했다"라고 그 함수를 호출한 커널 모듈에게 말해주지만 내부적으로는 page안에 그 조각을 계속갖고 있는 형태이다.

* 이렇게 하면 지속적으로 alloc, free의 overhead를 줄일 수 있다.

* 미리 하나의 페이지를 작은 object들로 미리 쪼개놓고 page를 그 object들로 꽉 채워서 사용할 수 있기 때문에 fragmentation(단편화)되는 문제점도 최소화할 수 있다.

### Slab의 내부 (꼭 필요한 건 아님)

* 이렇게 구분되어 있는 이유는 slab은 내부적으로 이미 할당을 해놓고 요청이 들어오면 주는데 이런게 kmem_cache 당 너무 많아지면 전체적으로 메모리가 부족해질 수 있으니 메모리가 부족하면 할당해놓은 object들을 운영체제가 회수해 간다. 그러면 slab_empty 쪽에는 아무것도 없으니 여기부터 회수해 간다. 

* slab 하나에는 하나 이상의 page가 있을 수 있다.

* page안에는 8개 object가 들어갈 수 있도록 미리 쪼개놓음.

#### kmem_Cache
* 자주 할당되는 object를 위한 slab allocator

* 예를들어 위에서 언급한 512byte짜리 object를 자주 할당하는 놈으로 하나의 kmem_cache entry를 만든다.

#### slab_full
* page 각각이 8개의 object로 구성이 된다고 가정을 했는데 그 8개가 다 사용된 경우

#### slab_empty
* page를 8개로 나누었는데 할당된 object가 하나도 없는 page를 모아 놓은 곳

#### slab_partial
* 일부는 사용되고 있고, 일부는 사용되고 있지 않은 page들을 모아 놓은 곳

### Cache Entry (kmem_cache entry를 만들어줌)

## Linked List
* linux가 제공하는 linked-list는 list_head라는 구조체에서 모두 조작할 수 있다.

* linked List를 다루기 위한 kernel 함수에는 locking에 대한 내용이 없다.

### list_head

* 임의의 구조체에 대한 pointer type을 선언할 수 없기 때문에 list_head에 대한 pointer로서 선언되어 있음. 즉, 우리가 임의의 선언한 구조체의 시작 주소들을 가리키진 않다.

## Spinlock

* Race condition: 서로 다른 context가 + operation을 했을 때,
메모리에 있는 data를 레지스터로 가져오고 레지스터에서 증가시키고 다시 메모리에 저장을 하는데, 그 사이에 다른 context에 있는 + operation이 들어옴으로써 아직 더한 결과를 메모리에 쓰기전에 또 메모리에서 레지스터에서 가져와서 더하고 그 결과값을 넣는 것이 서로 중복적으로 발생함으로써 더하기가 덜 실행되는 결과

* spin: 계속 lock이 있는지 보다가(cpu를 계속 pulling하면서 보다가) lock이 풀리는 순간 lock을 걸고 들어가고 다른 놈들은 계속 spin하면서 critical section에 들어가는 걸 기다리는 기법

## Adding Delay(Kernel 내부에 delay 넣기)

### udelay, mdelay
* 이 함수들은 내가 명시한 시간 값을 계속해서 확인한다. (권한을 얻을 때까지 계속 확인한다.)

* sleep()같은 함수는 process가 running state에서 waiting states나 block state cpu 자원을 양보하도록 되어 있다. 그런데 이 함수들은 계속해서 내부적으로 시간 값을 busy-waiting하면서 확인하고 그 시간이 지났을 때 return하는 함수이다.

* (device driver는 h/w와 이야기를 주고 받는 software인데 h/w에 어떤 명령을 주었을 때 그것이 끝났는지 확인 할 때, 가끔 경험적으로 몇 millisec이 지나는 동안 무조건 기다렸다가 그다음 동작을 해야 h/w적으로 그 시간 동안에 초기화라든지 어떤 operation이 끝난다라고 가정하는 경우가 있다.)

### msleep
* kernel안 함수인데 마치 sleep 함수처럼 process context를 재운다.

* interrupt handler는 process같은 context가 아니기 때문에 running state, waiting state를 따르는 context가 아니다. 그래서 interrupt handler 안에서는 sleep하는 함수를 부를 수 없다. 

* 하지만 system call을 해서 kernel안으로 진입한 process의 context인 경우 부를 수 있다. < 중요!










가상화 메모리, interrupt handler, critical section

Paging: 프로세스가 차지하는 물리적 메모리 공간이 비연속적이 되도록 허용하는 메모리 관리 기법을 말한다.

단편화: 메인 메모리에 빈번하게 기억 장소가 할당되고 반납됨에 따라 메모리 공간이 작은 조각 공간으로 나뉘게 될 경우, 사용 가능한 메모리가 충분함에도 불구하고 메모리 할당이 불가능한 상태를 말한다.

내부 단편화: 주기억장치 내의 실행 프로그램보다 사용자 영역(할당된 영역)이 커서 메모리 할당 후 사용되지 않고 남아있는 공간

외부 단편화: 주기억장치 내의 사용자 영역보다 실행 프로그램이 커서 프로그램이 메모리에 할당되지 않고 남아있는 공간

race condition: 공유 자원에 대해 여러 프로세스가 동시에 접근을 시도할 때, 타이밍이나 순서 등이 결과값에 영향을 줄 수 있는 상태

deadlock: 두 개 이상의 작업이 서로 상대방의 작업이 끝나기 만을 기다리고 있기 때문에 결과적으로 아무것도 완료되지 못하는 상태

