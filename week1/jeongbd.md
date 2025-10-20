
# JVM 밑바닥 까지 파헤치기 2장 
## Java Runtime Data Area

자바 가상 머신은 자바 프로그램을 실행하는 동안 메모리를 아래의 그림과 같이 몇 개의 데이터 영역으로 나누어 관리하며 각 영역은 목적과 생성 삭제 시점이 존재 한다. 

### Program Counter Register 
프로그램 카운터(PC) 레지스터는 작은 메모리 공간으로, 현재 실행 중인 스레드가 어느 바이트코드 위치까지 실행했는지를 표시하는 역할을 한다.

자바 가상 머신에서 여러 스레드가 동시에 실행될 때, CPU 코어는 각 스레드를 순서대로 번갈아 실행하는데 이 때 스레드가 실행 도중 멈추고 다른 스레드로 전환되면, 중단된 위치를 정확히 기억해야 하기 때문에 각 스레드는 고유한 PC 레지스터를 가지고, 서로 영향을 주지 않고 독립적으로 실행 위치를 관리한다.

### Method Area
메서드 영역은 자바 힙 처럼 모든 스레드가 공유 하는 영역이다. 메서드 영역은 가상 머신이 읽어들인 타입 정보, 상수, 정적 변수 JIT 컴파일러가 코드 캐시 등을 저장하는 공간이다. 

#### Runtime Constant Pool 
클래스 버전, 필드, 메서드, 인터페이스 등 클래스 파일에 포함된 설명 정보에 더해 컴파일 타임에 생성된 다양한 리터럴과 심벌 참조가 저장된다.  
컴파일 후 생성되는 .class 파일 내부에 있는 상수 풀(Constant Pool)의 정보들을 JVM이 이 클래스를 메모리에 로드할 때 그것을 읽어 메서드 영역에 있는 런타임 상수 풀(Runtime Constant Pool)으로 복사한다. 실제 java파일을 컴파일 후 생성되는 클래스 파일에서 상수 풀의 정보를 확인할 수 있다.

```bash
public class com.example.demo.ch01.ClassConstantPool
  minor version: 0
  major version: 65
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #25                      // com/example/demo/ch01/ClassConstantPool
  super_class: #2                      // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 3
Constant pool:
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   #3 = NameAndType        #5:#6          // "<init>":()V
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = String             #8             // Hello world
   #8 = Utf8               Hello world
   #9 = Fieldref           #10.#11        // java/lang/System.out:Ljava/io/PrintStream;
  #10 = Class              #12            // java/lang/System
  #11 = NameAndType        #13:#14        // out:Ljava/io/PrintStream;
  #12 = Utf8               java/lang/System
  #13 = Utf8               out
  #14 = Utf8               Ljava/io/PrintStream;
  #15 = InvokeDynamic      #0:#16         // #0:makeConcatWithConstants:(Ljava/lang/String;I)Ljava/lang/String;
  #16 = NameAndType        #17:#18        // makeConcatWithConstants:(Ljava/lang/String;I)Ljava/lang/String;
  #17 = Utf8               makeConcatWithConstants
  #18 = Utf8               (Ljava/lang/String;I)Ljava/lang/String;
  #19 = Methodref          #20.#21        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #20 = Class              #22            // java/io/PrintStream
  #21 = NameAndType        #23:#24        // println:(Ljava/lang/String;)V
  #22 = Utf8               java/io/PrintStream
  #23 = Utf8               println
  #24 = Utf8               (Ljava/lang/String;)V
  #25 = Class              #26            // com/example/demo/ch01/ClassConstantPool
  #26 = Utf8               com/example/demo/ch01/ClassConstantPool
  #27 = Utf8               Code
  #28 = Utf8               LineNumberTable
  #29 = Utf8               main
  #30 = Utf8               ([Ljava/lang/String;)V
  #31 = Utf8               SourceFile
  #32 = Utf8               ClassConstantPool.java
  #33 = Utf8               BootstrapMethods
  #34 = String             #35            // \u0001\u0001
  #35 = Utf8               \u0001\u0001
```
### Heap 
애플리케이션이 사용할 수 있는 가장 큰 애플리케이션 메모리 자바 힙은 모든 스레드가 공유한다. 가상 머신이 구동될 때 만들어지며 모든 객체 인스턴스와 배열은 힙에 할당 된다. 

#### Java 7 Java 8 의 변화
Java 7 HotSpot VM의 메서드 영역을 Permanent Generation에 구현한 것은 자바 애플리케이션들이 메모리 오버플로를 겪을 가능성이 커졌다. Permanent Generation의 최대 크기는 -XX:MaxPermSize제한되었으며 Permanent 영역은 힙 영역의 일종으로 취급되고 있었다.  Java 8 HotSpotVM에 와서는 Permanent Generation이라는 개념을 완전히 지우고 네이티브 메모리에 메타스페이스를 구현 했다. 힙에서 분리 함으로써 필요시 물리적 메모리 한도 내에서 자동으로 확장이 가능하다. 

### Virtual Method Stack
자바 가상 머신 스택은 스레드와 운명을 같이 한다. 각 메서드가 호출될 때마다 자바 가상 머신은 스택 프레임을 생성한다. 지역 변수 테이블, 피연산자 스택, 동적 링크, 메서드 반환값등의 정보를 저장한다. 

지역 변수 테이블에는 데이터 타입, 객체 참조, 반화 주소 타입을 저장 한다. 지역 변수 테이블에서 이 데이터 타입들을 저장하는 공간을 지역 변수 슬롯이라고 한다. 지역 변수 테이블을 구성하는데 필요한 데이터 공간을 컴파일 과정에서 할당된다. 

#### 스택 메모리 영역에서 두가지 에러 
스래드가 요청한 스택 깊이가 가상 머신이 허용하는 깊이 보다 크면 StackOverflowError가 발생하고 스택 용량을 동적으로 확장할 수 있는 자바 가상 머신에서는 스택을 확장하려는 시점에 여유 메모리가 충분하지 않으면 OutOfMEmoryError 가 발생한다. 

### Native Method Stack 
네이티브 메서드 스택은 자바 가상 머신 스택과 매우 비슷한 역할을 한다.  
가상머신 스택은 자바 메서드를 실행 할때 사용하고 네이티브 메서드 스택은 네이티브 메서드를 실행할 때 사용한다.

---

## 객체의 생성과 접근
자바 가상 머신이 new에 해당하는 바이트 코드를 만나면 이 명령의 매개변수가 상수 플 안의 클래스를 가리키는 심벌 참조인지 확인한다. 이 심벌참조가 뜻하는 클래스가 로딩-해석-초기화 되었는지 확인후 로딩이 완료된 클래스라면 새 객체를 담을 메모리를 할당한다.  **메모리 할당이란 자바 힙에서 특정 크기의 메모리 블록을 잘라 주는 일**이라고 볼 수 있다. 

### 객체의 생성 
> **Bump the pointer**  
 자바 힙이 규칙적일 경우에는 사용중인 메모리는 한 쪽에 여유 메모리는 반대편에 자리해서 포인터가 두 영역 경계인 가운데 지점을 가리키게 되고 할당 시 포인터를 여유 공간으로 객체 크기 만큼 이동시키는 방식을 사용한다. 

> **Free list**  
자바 힙은 규칙적이지 않을 경우사용 메모리와 여유 메모리가 규칙적이지 않고 뒤섞여 있기 때문에 포인터를 밀쳐내는 것이 간단하지 않다. 자바 머신은 가용 메모리 블록들을 목록으로 따로 관리하며 객체 인스턴스를 담기에 충분한 공간을 찾아 할당 후 목록을 갱신 하는 방식을 사용한ㄴ다.. 

위 두 가지 중에 어떤 방식을 쓸 것인가는 힙이 규칙적인가의 여부에 달려있으며 또 힙이 규칙적인가는 가비지 컬렉터가 컴팩트 모으기를 할 수 있는가 없는가에 달려있다.  

### 안전한 메모리 할당 
가상 머신에서 객체 생성은 매우 빈번하게 일어나고 멀티스레딩환경에서 여유 메모리의 시작 포인트의 워치를 수정하는 단순한 일도 스레드 안전하지 않고 여러 스레드가 동시에 객체를 생성하려고 할 때 문제가 생길 수있다. 

메모리 할당을 동기화하는 방식과 스레드 마다 다른 메모리 공간을 할당 받는 방법이다. 스레드 각각이 자바 힙내에 작은 크기의 전용 메모리를 미리 할당 받아 놓는데 이런 메모리를 스레드 로컬 할당 버퍼라고 한다. 각 스레드는 로컬 버퍼에서 메모리를 할당 받아서 사용하다가 버퍼가 부족해지면 그때 동기화해서 새로운 버퍼를 할당 받는 방식이다. 

메모리 할당이 끝났으면 가상머신은 할당 받은 공간을 객체 헤더를 제외하고 0으로 초기화한다. 스레드 로컬 할당 버퍼를 사용한다면 초기화는 TLAB 할당 시에 미리 수행한다. 자바 코드에서 객체의 인스턴스 필드를 초기화 하지 않고 사용할 수 있는 것이 이 단계 과정 덕분이다. 모든 필드가 자연스럽게 각 데이터 타입에 해당하는 0 값을 담고 있게 된다. 


### 객체의 접근 
자바 프로그램은 스택에 있는 참조 데이터를 통해 힙에 있는 객체들에 접근해서 이를 조작한다. 자바 가상 머신 명세에서는 힙에서 객체의 정확한 위치를 알아내어 접근하는 방법은 규정하고 있지 않기 때문에 객체에 접근하는 것은 가상머신에서 구현하기 나름이며 주로 핸들과 다이렉트 포인터 방식을 사용해서 구현한다. 


#### 핸들 방식 
자바 힙에는 핸들 저장용 풀이 별도로 존재한다. 즉 핸들 참조 객체의 핸들 주소가 저장되고 핸들에는 다시 객체의 인스턴스 데이터 타입 데이터, 구조 등의 주소 정보가 담길 것이다. 핸드 방식은 안정적인 핸들 주소가 저장된다. 핸들을 이용하면 객체의 위치가 바뀌는 상황에서도 참조 자체에는 손댈 필요가 없다. 대신 핸들 내의 인스턴스 데이터 포인터만 바꾸면 된다. 

##### 다이렉트 포인터 
자비 힙에 위치한 객체에서 인스턴스 데이터 뿐만 아니라 타입 데이터에 접근하는 길도 제공해야 한다. 스택의 참조에는 객체의 실제 주소가 바로 저장되어 있다. 다이렉트 포인터 방식 장점은 속도로 핸들을 경유하는 방식에 비해 오버헤드가 없다. 자바에서는 다른 객체에 접근할 일이 아주 많기 때문에 오버헤드도 실행 시간에 영향을 준다. 

---

## 객체 메모리 구조
자바 가상 머신은 객체를 세부분으로 나누어서 힙에 저장한다.  

### Object Header 
객체 헤더에 두 유형의 정보를 담는다. 객체 자체의 런타임 데이터

#### Mark word 
객체 자체의 런타임 데이터를 저장한다. 해시코드, GC세대 나이, 락 상태 플래그, 스래드가 점유하고 있는 락, 편향된 스레드와 아이디 편향된 시각의 타임 스탬프 등을 담고 있다. 

#### Klass word 
객체의 클래스 워드에는 객체의 클래스 관련 메타데이터를 가리키는 클래스 포인터가 저장되며 이 가상 머신은 이 포인터를 통해 특정 객체가 어느 클래스의 인스턴스인지 런타임에 알 수 있다. 모든 가상 머신 구현이 클래스 포인터를 객체 헤더에 저장하지는 않는다. 

#### Array length 
배열 길이도 객체 헤더에 저장하는데 객체 헤더의 메타 데이터로부터 자바 객체의 크기를 얻지만 객체 헤더에 저장되는 객체 타입은 배열의 원소 타입이기 때문에 배열의 길이를 알아야 배열 객체가 차지하는 메모리 크기를 제대로 계산할 수 있다. 

### instance data 
객체가 실제로 담고 있는 정보, 다양한 타입의 필드 관련 내용, 부모 클래스 유무, 부모 클래스에서 정의한 모든 필드가 이 부분에 기록된다.

### padding 
핫스팟 가상 머신의 자동 메모리 관리 시스템에서 객체의 시작 주소는 반드시 8바이트 정수배여야 한다.

> [저우즈밍, JVM 밑바닥까지 파헤치기, 인사이트(2024)](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=338394581)
