# Context Switching

## 정의 및 개념

현재 프로세스의 상태를 저장하고 처리가 필요한 다른 프로세스로 프로세서의 처리 대상을 이전하는 행위.

(=프로세서가 처리할 프로세스를 바꾸는 일)

본래 의미상으로는 프로세스를 바꾸는 것이 맞지만, 정확히는 프로세스에 대한 정보가 기술되어있는 블록(Process Control Block : PCB)를 바꾸는데, 이 블록을 Context라고 하기 때문에 Context Switching이라고 부름(블록 + 블록을 관리하기 위한 여타 정보를 교환함).

언제 어느때나 발생할 수 있으며, 발생할 경우 아래 2가지 의문이 발생

1. 어떤 이벤트가 context switching을 초래하는가? (아래 발생시점 및 원인에서 설명)
2. context switching을 완수하기 위해선 OS는 어떤 정보를 저장 및 관리해야 하는가?
    
    ![image.png](image.png)
    
    1. Process Identification : 프로세스 식별 정보
        
        PID(Process ID) : 각 프로세스마다 부여된 고유 번호
        
        PPID(Parent Process ID) : 이 프로세스를 생성한 부모 프로세스의 ID
        
        UID(User ID) / GID(Group ID) : 프로세스의 소유자와 그룹의 식별자
        
        → 프로세스를 생성, 추적, 종료, 접근제어, 자원할당 등 관리하는데 사용하는 정보
        
    2. Processor State Information : 이 프로세스를 실행중일 때의 프로세서 상태를 저장하는 정보
    
    →현재 프로세스가 중단되었다가 다시 실행되었을 때, 어디서부터 어떻게 실행해야 하는지를 담음
    
    CPU Register 
    
    GPRs(General Purpose Registers) : 임시 데이터 저장소
    
    e.g., EAX, EBX, ECX ,,, etc.
    
    PC(Program Counter) : 다음에 실행할 명령어 주소
    
    SP(Stack Pointer) : 이 프로세스의 Memory stack의 최상단 지점
    
    FP(Frame Pointer) : 지역변수, 매개변수 접근에 사용하는 포인터
    
    PSW(Program Status Word) : 연산 결과 플래그를 담는 레지스터
    
    Zero flag : 연산 결과가 0인지
    
    Privilege/Mode bit : 프로세스가 어떤 모드에서 운용되었는지
    
    Interrupt enable flag : 인터럽트 허용 여부
    
    ![image.png](image%201.png)
    
    1. Process Control Information : 프로세스 관리 및 스케줄링을 위한 정보
        
        Process Status : ready, blocked, running 등 프로세스의 상태
        
        Process Priority : 프로세스의 우선순위. 스케줄링 결정에 사용
        
        Scheduling Information : 스케줄링 큐 위치, 최근실행시각 등 스케줄링에 사용되는 기타 정보
        
        Resource Allocaiton Information : 메모리 블록 위치, 입출력장치, 파일목록 등 프로세스가 점유/사용하는 자원 정보
        
        권한, 계정, 입출력 및 통신상태 등 기타 제어정보가 모두 포함됨
        

## 과정

![image.png](image%202.png)

위는 Context switching의 과정을 나타낸 모식도. 그림 그대로가 과정.

kernel/sched.c 에 정의된 실제 context_switching() 함수의 정의 부분. 이 구현 내용에 따라 context switching이 수행됨.

![image.png](image%203.png)

---

전달인자:

runqueue_t *rq : 실행 대기중인 프로세스 집합

task_t *prev : 현재까지 실행중이던 프로세스의 PCB

task_t *next : 새롭게 실행할 프로세스의 PCB

---

Line no. 1374 ~ 1388

현재까지 진행중이던 프로세스의 메모리 참조에서 곧 수행할 프로세스의 메모리 참조로 위치를 이동하는 연산.

만약 앞으로 실행할 프로세스의 주소공간 위치가 NULL일 경우(=커널 스레드일 경우), 이전 프로세스가 사용하던 메모리의 주소를 받아 사용

이유  :

⇒ Linux에선 1개의 가상 주소 공간에 유저 영역 + 커널 영역을 합쳐 매핑함

e.g., 32비트에서 메모리에서 상위 1GB는 커널공간, 하위 3GB는 유저 공간

= 모든 프로세스는 모두 같은 커널 영역을 가리키기 때문에, 이전 프로세스가 사용하던 메모리의 주소를 그대로 받으나 새로 만드나 어차피 가리키는 커널 공간의 위치는 똑같음.

## 발생시점 및 원인

### Context Switching을 일으키는 이벤트

![image.png](image%204.png)

과정 별 구체적은 사유는 아래와 같음

- Interrupt : 인터럽트 핸들러로 context switching이 발생하는 과정.
    1. Clock Interrupt : 프로세서가 할당한 처리 시간이 모두 경과되었을 경우 발생하는 인터럽트. 이 경우, 실행중이던 프로세스는 할당 시간이 만료되어 Ready 상태로 전이하고, 다른 프로세스로 Dispatch됨
    2. I/O Interrupt : I/O 인터럽트에 대한 처리가 필요할경우 인터럽트 핸들러로 Context swtiching이 발생함. 만약 이 동작이 하나 이상의 프로젝트가 대기중이던 인터럽트일 경우, 대기중이던 모든 프로세스를 Blocked 상태에서 Ready 상태로 옮김. 이후 현재 실행중인 프로세스를 마저 실행할지, 방금 Ready 상태로 올라온 프로세스로 Context switching을 수행할지 우선순위를 기반으로 결정.
    3. Memory Fault : 만약 프로세스가 현재 메모리에 로드되어있지 않은 메모리에 대한 참조를 요청한 경우 Page Fault가 발생. 이 경우, OS는 디스크에서 해당 페이지/세그먼트를 메모리로 가져오도록 I/O 명령을 내림. 이 과정이 연산 속도에 비해 매우 길고 느리기 때문에, 해당 프로세스를 Blocked 상태로 전환하고 다른 프로세스를 시행함(이 과정에서 Context Switch 발생). 필요한 데이터가 메모리에 모두 로딩되면. 위 2.와 같은 로직으로 Ready 상태로 올라
- Trap : OS는 trap을 발생시킨 오류나 예외가 얼마나 치명적인지를 판단. 만약 심각한 문제라면, 현재 실행중인 프로세스는 Exit 상태로 전이되고 다른 프로세스로 Context Switching 발생
(= 더 이상 실행이 불가능하다는 의미)
만약 심각하지 않다면 이후 조치는 OS의 종류와 성격에 따라 달라짐.
    - “심각하다(Fatal)” 의 기준 : 복구가 불가능한지를 판별
        - e.g.,
            - 잘못된 명령어 실행(Illegal Instruction)
            - 허용되지 않은 메모리 접근(Segmentation Fault, Invalid Memory Access)
            - 0으로 나누기(Divide by zero)
                - 수학적으로 결과가 정의되지 않은 연산이기 때문에, 컴퓨터는 이를 표현할 방법이 없어 종료하는 식으로 회피 처리. 거의 모든 현대 운영체제는 이를 치명적인 오류로 간주함.
            - Stack Overflow
- Supervisor Call : System call이 발생했을 경우에 해당. 유저가 직접 OS의 서비스를 요청할 경우 발생함.
    - system call 자체는 context switching을 유발하지 않지만(getpid(), time(), getuid()등 발생시키지 않는 종류가 더 많음), I/O 처리가 즉시 되지 않는 system call을 실행하면 blocked 상태로 전환됨. 이때 context switching이 발생하는 것.

## 오버헤드

위 과정에서 설명한 Context Switching의 모든 절차는 CPU 효율 관점에서 보면 전부 다 오버헤드.

⇒ 실제 연산을 수행할 수 있는 시간임에도 PCB를 저장 및 복원하는 과정에 연산 능력을 사용하고 있기 때문

하지만 싱글코어 CPU에서도 멀티태스킹을 지원하고, I/O 입출력 대기 등 한 프로세스가 대기하고 있는 동안 다른 프로세스를 수행하여 CPU를 낭비시키지 않기 때문에, 위와 같은 오버헤드가 발생해도 사용하는 편이 이득인 trade-off 관계.

이 오버헤드는 SW적으로 복잡도나 코드 최적화, 빌드 옵션 등을 이용해서 시간을 줄일 수 있지만, CPU의 속도나 캐시/메모리 대역폭 등 HW적인 부분에서도 많은 영향을 받음

## 프로세스/스레드 관점에서 Context switching

### 프로세스 vs 스레드

프로세스는 독립된 메모리 공간을 소유하는 프로그램의 실행 단위. 각 프로세스는 전부 다 다른 메모리 공간을 점유함.

스레드는 프로세스 내부에서 실행 흐름만을 분리한 경량 실행 단위. 한 프로세스 안에서 분리된 스레드들은 stack, PC, register를 제외한 거의 모든 메모리 공간을 공유함.

### 각 단위별 Context switching

프로세스는 각기 독립된 메모리 공간을 점유하고 있기 때문에, MMU의 페이지 테이블도 완전히 전환해야 함. 따라서 거의 무조건적으로 cache cold에 상태에 놓이므로, 오버헤드가 커서 필연적인 성능 저하를 일으킴

스레드는 대부분의 메모리 공간을 공유하기 때문에, MMU의 추가 조작이 필요하지 않음. 따라서 같은 프로세스 안에서 이루어지는 Thread Context Switching은 TLB flush, 캐시 무효화, 주소공간 전환 등이 필요하지 않아 cache hot을 유지해 상대적으로 적은 오버헤드가 발생함.

TLB(Translation Lookaside Buffer) : MMU 안에 내장된 고속 캐시. 가상 주소를 실제 주소로 매핑하는 데 사용되며, 자주 참조하는 페이지 테이블 엔트리를 저장해두는 캐시.  TLB의 hit율이 높을수록 메모리 참조가 빠름.