- 64p 물리적으로도 연속된 메모리 공간을 사용하도록 구현한다
	- 리눅스사용시 가상메모리를 사용하게되면 어플리케이션은 논리적인 메모리만 알게되기 때문에 논리메모리가 연속된다고 봐야할것 같다.
- 64p 자바 힙은 크기를 고정할 수도, 확장할 수도 있게 구현할 수 있다.
	- -Xms: 초기 시작 메모리 크기
	- -Xmx: 최대 메모리 크기
	- jvm구동시 Xms크기만큼 메모리를 예약(reserve)함
		- 가상메모리 주소만 할당받은 상태로 실제 메모리가 소모되지 않음
		- 메모리를 실제로 사용시 page fault가 발생하여 Xms크기만큼 메모리가 실제 사용(commit)됨
	- Xmx크기만큼 메모리가 추가로 사용(commit)될 수 있음
- 66p 심볼 참조
	- 컴파일시점에 해당값의 위치를 알수 없어 참조값으로만 저장되어 있다가, 실행시점에 실제값에 접근 가능해짐
	- 컴파일 시점에 user 메모리주소를 알 수 없어(힙에 있으니까) 심볼 참조가 됨
```java
String name = user.getName();
```
- 73p 락 플래그
	- 락 플래그가 01(잠기지 않음)인 상태에서 존재하는 데이터(해시코드, 객체나이)는 락이 걸렸을때 어떻게 되는가?
	- 락을 걸을 스레드의 스택으로 옮겨진다. 해당객체는 락을건 스레드만 접근이 가능하기때문에 해당데이터를 다른스레드 gc에서 접근할 수 없어진다.
	- 편향락
		- 실제 synchronized과 같은 무거운 작업을 수행하지 않으며 편향을 획득한 스레드 아이디와 같은지만 가볍게 체크하는 상태
		- 편향 스레드가 아닌 스레드가 접근시 편향이 없어지면 경량/중량 락을 점검하게됨
	- 경량락
		- 마크워드와 비슷한 lock record생성하여 락을 획득하고자 하는 객체의 마크워드와 교체를 주기적으로 시도함
		- 교체가 성공하면 다른스레드에서 락을 풀어줬다는 의미이므로 락을 획득하는 효과임
		- 곧 내가 락을 획득 가능하다고 생각할때 시도함
	- 중량락: 그냥 진짜 락
- 88p 2-8 참조테스트
```java
StringBuilder builder = new StringBuilder("컴퓨터").append(" 소프트웨어");  
String str1 = builder.toString();  
String str2 = builder.toString();  
String intern1 = str1.intern();  
String intern2 = str2.intern();  
System.out.println(str1 == str2); // false
System.out.println(intern1 == intern2); // true
System.out.println(intern1 == str1); // true
System.out.println(intern1 == str2); // false
```
