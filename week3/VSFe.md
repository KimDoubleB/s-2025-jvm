## 클래식 가비지 컬렉터 (Classic Garbage Collectors)

JVM은 벤더와 버전에 따라 다양한 Garbage Collector(GC)를 제공하며, 실행 옵션을 통해 원하는 컬렉터를 선택할 수 있다.

------

### Serial Collector

가장 단순하고 오래된 형태의 GC이다. 단일 스레드(single-thread)로 동작하며, 모든 단계에서 **Stop-The-World(STW)** 가 발생한다.

이 방식은 구조가 단순하고 메모리 오버헤드가 작지만, GC가 수행되는 동안 애플리케이션 스레드가 모두 중단되기 때문에 STW 시간이 상대적으로 길어진다.
 단일 코어 환경이나 STW가 문제되지 않는 배치성 작업에서 주로 사용된다.

------

### ParNew Collector

`Serial Collector`와 알고리즘은 동일하지만, **멀티 스레드(multi-thread)** 로 동작하도록 확장된 버전이다.
 해당 컬렉터는 **New Generation** 영역의 수집을 담당하며, **CMS(Concurrent Mark-Sweep)** 와 결합하여 동작한다.
 현재는 독립적으로 사용할 수 없으며, CMS 전용 전처리 단계로만 존재한다.

------

### Parallel Scavenge Collector

`Parallel Scavenge`는 **Mark-Copy** 기반의 멀티 스레드 컬렉터로, 주요 목표는 **처리량(Throughput)** 극대화에 있다.
 즉, 개별 일시 정지 시간을 최소화하기보다 전체 애플리케이션 실행 효율을 높이는 데 초점을 맞춘다.

처리량(Throughput)은 다음 식으로 정의된다.

```
Throughput = (Application Run Time) / (Application Run Time + GC Time)
```

대표적인 설정 옵션은 다음과 같다.

- `-XX:MaxGCPauseMillis` : GC로 인한 일시 정지 시간이 해당 값(밀리초)을 넘지 않도록 노력한다.
   단, 이 값을 짧게 설정할수록 회수 효율이 떨어질 수 있음을 유의해야 한다.
- `-XX:GCTimeRatio` : 전체 애플리케이션 실행 시간 중 GC가 차지하는 비율을 설정한다.
   기본값은 99이며, 이는 GC 시간이 전체의 약 1% 이하임을 의미한다.
- `-XX:+UseAdaptiveSizePolicy` : `New Generation` 크기(`-Xmn`), Eden과 Survivor 비율(`-XX:SurvivorRatio`), Old 영역 승격 임계치(`-XX:PretenureSizeThreshold`) 등을 자동으로 조정한다.
   JVM이 모니터링 지표를 기반으로 정지 시간과 처리량의 균형을 동적으로 맞춘다.

------

### Serial Old Collector

`Serial Collector`의 **Old Generation** 버전이다. **Mark-Compact** 알고리즘을 사용하며, 주로 **Client VM**이나 **CMS의 fallback(압축 단계 필요 시)** 용도로 사용된다.

------

### CMS (Concurrent Mark-Sweep) Collector

**CMS(Concurrent Mark-Sweep)** 컬렉터는 이름 그대로 객체를 “표시(Mark)”하고 “쓸어내는(Sweep)” 과정을 애플리케이션 스레드와 동시에 수행하도록 설계되었다.
 이 컬렉터의 핵심 목표는 **GC로 인한 일시 정지 시간을 최소화**하는 것이다. 사용자 경험이 중요한 애플리케이션(예: 데스크톱 애플리케이션, 브라우저 등)에 적합하다.

CMS는 **Mark-Sweep** 알고리즘을 기반으로 동작하며, 수행 과정은 다음 네 단계로 구성된다.

1. **Initial Mark (STW)** : GC Roots가 직접 참조하는 객체를 1차로 표시한다.
2. **Concurrent Mark** : 사용자 스레드와 동시에 객체 그래프를 탐색하여 참조 가능한 객체를 표시한다.
3. **Remark (STW)** : 동시 표시 중 변경된 참조를 다시 수정하여 일관성을 보장한다.
4. **Concurrent Sweep** : 표시되지 않은 객체를 해제하며, 이 단계도 애플리케이션과 동시에 진행된다.

이 과정에서 “살아 있는 객체”는 이동하지 않기 때문에, 대부분의 단계가 애플리케이션과 병행 수행될 수 있다.

------

#### 단점

1. **프로세서 자원 의존성**
    CMS는 동시성을 확보하는 대신 CPU 자원을 많이 사용한다.
    기본적으로 CMS는 `(CPU 코어 수 + 3) / 4` 만큼의 GC 스레드를 생성하며, 이는 저사양 환경에서는 애플리케이션 성능 저하로 이어질 수 있다.
2. **Floating Garbage 발생**
    Concurrent 단계 이후 새로 죽은 객체는 Sweep 단계에서 수집되지 못한다.
    이런 객체들은 다음 GC 사이클에서 처리된다.
3. **메모리 여유 공간 요구**
    CMS는 동시 Sweep 단계에서도 애플리케이션이 정상 동작할 수 있도록 충분한 여유 공간이 필요하다.
    Old Generation이 거의 가득 차기 전에 GC를 트리거해야 한다.
4. **단편화(Fragmentation)**
    CMS는 **Mark-Sweep** 기반이므로 메모리 단편화가 발생한다.
    이 문제를 해결하기 위해서는 전역 압축(Full GC)을 수행해야 한다.

CMS는 이후 등장한 **G1**, **Shenandoah**, **ZGC** 등의 발전된 컬렉터들에 의해 대체되었으며, JDK 14에서 완전히 제거되었다.

### G1 컬렉터 (Garbage-First Collector)

`G1`은 *Garbage First*의 약자로, 기존 컬렉터들이 세대(Generation) 단위로 전체 영역을 수집하던 방식에서 벗어나 **리전(Region)** 기반의 수집 단위를 도입한 컬렉터이다.
 이는 “가장 효율적인 영역부터 먼저 회수한다”는 아이디어를 기반으로 하며, **Partial Collection(부분 회수)** 개념을 실현했다.

G1은 주로 서버 애플리케이션을 대상으로 설계되었으며, **지연 시간(Latency) 제어와 처리량(Throughput)의 균형**을 목표로 한다.

------

#### 기본 개념

G1은 힙(Heap)을 크기가 동일한 여러 개의 **Region**으로 나눈다.
 각 Region은 동적으로 **Eden**, **Survivor**, **Old Generation** 등의 역할을 가질 수 있으며, 필요에 따라 용도가 바뀐다.
 즉, 세대는 논리적으로만 구분되며, 물리적으로 연속되지 않는다.

또한 대형 객체(Region 크기의 절반 이상)는 **Humongous Region**이라 불리는 특별한 공간에 저장된다.
 Region 크기는 `-XX:G1HeapRegionSize` 옵션으로 지정할 수 있으며, 1MB~32MB 범위에서 2의 제곱 단위로 설정된다.

이 구조 덕분에 G1은 특정 세대 전체가 아니라 “회수 효율이 높은 Region들”만 선택하여 수집할 수 있다.
 이때 선택된 Region 집합을 **Collection Set(CSet)** 이라고 부른다.

------

#### 동작 원리

G1은 각 Region의 **Garbage Ratio(쓰레기 비율)** 을 추적하고,
 `(회수 가능한 메모리 크기) / (회수 소요 시간)` 값을 기반으로 우선순위를 계산한다.
 이후 사용자가 설정한 일시 정지 한도(`-XX:MaxGCPauseMillis`, 기본값 200ms)를 만족하는 범위 내에서
 “가장 효율적인 Region들”을 선택하여 수집한다.

------

#### 단계별 동작 과정

G1의 전체 GC 과정은 다음 네 단계로 이루어진다.

1. **Initial Mark (STW)**
    GC Roots가 직접 참조하는 객체를 표시하고, `TAMS(Top-at-Mark-Start)` 포인터를 기록한다.
    이 시점의 스냅샷을 기준으로 이후의 동시 표시가 수행된다.
    정지 시간은 매우 짧으며, G1이 “저지연 컬렉터”로 분류되는 이유 중 하나이다.
2. **Concurrent Mark**
    애플리케이션 스레드와 동시에 객체 그래프를 탐색하며, 살아 있는 객체를 표시한다.
    이후 Snapshot-at-the-Beginning(SATB) 알고리즘을 이용해 표시 도중 변경된 참조를 추적한다.
3. **Remark (STW)**
    Concurrent Mark 중 변경된 참조를 다시 확인하고, 표시 누락된 객체를 보정한다.
    처리량이 적기 때문에 짧은 시간 내에 완료된다.
4. **Cleanup / Copy (STW)**
    회수 가치가 높은 Region들을 선별하고, 생존 객체를 다른 빈 Region으로 복사한다.
    복사가 끝난 Region은 비워져 재사용된다.

이 과정을 반복하면서 G1은 짧은 STW 구간을 여러 번 분산시켜 전체 일시 정지를 예측 가능하게 만든다.

------

#### 내부 구현 세부 사항

1. **기억 집합(Remembered Set, RSet)**
    Region 간 객체 참조를 추적하기 위한 구조이다.
    GC가 전체 힙을 스캔하지 않고도 필요한 참조 관계만 확인할 수 있도록,
    각 Region은 “외부에서 자신을 참조하는 카드 인덱스 목록”을 해시 테이블 형태로 유지한다.
    이 구조는 메모리를 더 사용하지만, STW 시간을 크게 줄여준다.
2. **스냅샷-기반 동시 표시(SATB)**
    GC 시작 시점의 객체 그래프를 기준으로 한 스냅샷을 유지하며,
    이후 변경된 참조를 별도로 기록한다.
    이를 통해 동시 표시 중에도 애플리케이션 스레드가 안전하게 객체를 수정할 수 있다.
3. **정지 시간 예측 모델**
    G1은 각 Region의 회수 비용과 이득을 지속적으로 샘플링하며,
    **감쇠 평균(Decaying Average)** 기법을 사용해 최신 트렌드에 가중치를 둔 예측을 수행한다.
    이를 통해 `MaxGCPauseMillis` 설정값 내에서 회수 가능한 최적의 Region 조합을 결정한다.

## 저지연 가비지 컬렉터 (Low-Latency Garbage Collectors)

과거에는 메모리 사용량 절감과 처리량 극대화가 주요 목표였지만, 오늘날에는 레이턴시가 가장 중요한 지표로 평가된다. 특히 서버 애플리케이션이나 실시간 처리를 요구하는 서비스에서는 긴 일시 정지 시간(STW)이 곧 서비스 중단으로 이어질 수 있기 때문이다.

- CMS 이전의 모든 컬렉터는 전 단계가 Stop-The-World였다.
- G1은 SATB 기반의 **동시 표시(Concurrent Marking)** 로 STW 시간을 줄였다.
- Shenandoah와 ZGC는 대부분의 단계를 **애플리케이션 스레드와 완전히 동시에(concurrent)** 수행한다. 이때 STW는 GC 루트 초기 표시와 최종 참조 보정 단계에서만 잠시 발생하며, 이 시간은 힙 크기와 무관하게 거의 일정하다.

이러한 특징 덕분에 Shenandoah와 ZGC는 **저지연(ultra-low pause) GC**로 분류된다.

------

### Shenandoah GC

**Shenandoah**는 Red Hat에서 개발한 GC로, G1의 설계를 계승하면서도 **동시 객체 이동(Concurrent Evacuation)** 을 지원하도록 확장한 버전이다.  JDK 12에서 정식 포함되었으며, OpenJDK 기반 배포판에서 기본적으로 활성화할 수 있다.

Shenandoah는 G1처럼 힙을 **Region 단위**로 관리하지만, Evacuation조차도 **Concurrent** 단계에서 수행한다.  즉, 객체가 이동하는 동안에도 애플리케이션 스레드가 계속 실행될 수 있다.

------

#### 동작 단계

1. **Initial Mark (STW)**
    GC Roots로부터 직접 참조되는 객체를 표시한다.
    이 단계는 매우 짧은 정지 구간이며, GC Root의 수에 비례한다.
2. **Concurrent Mark**
    애플리케이션 스레드와 동시에 객체 그래프를 탐색하여 도달 가능한 객체를 표시한다.
    이 단계에서 새로 생성된 객체는 별도의 기록 없이 자동으로 “살아있는 객체”로 간주된다.
3. **Final Mark (STW)**
    표시가 완료되면, 회수할 Region 집합(Collection Set)을 결정한다.
4. **Concurrent Cleanup**
    살아있는 객체가 전혀 없는 Region을 비운다.
5. **Concurrent Evacuation**
    Collection Set 내의 살아있는 객체를 빈 Region으로 복사한다.
    애플리케이션 스레드와 동시에 실행되며, 이 단계가 Shenandoah의 핵심이다.
6. **Initial Update References (STW)**
    모든 스레드의 Evacuation이 완료되었음을 보장하기 위해 짧은 동기화 구간을 가진다.
7. **Concurrent Update References**
    실제로 모든 참조 포인터를 새 객체 주소로 갱신한다.
    이 작업은 선형 탐색 방식으로 진행된다.
8. **Final Update References (STW)**
    GC Root의 참조만 갱신한 뒤, 남은 Region을 청소한다.

------

#### 내부 기술

- **포워딩 포인터(Forwarding Pointer)**
   객체가 새로운 Region으로 이동할 때, 원본 객체의 헤더에 새 주소를 기록한다.
   다른 스레드가 아직 옛 주소를 참조하더라도, 읽기 시점에 포인터를 따라가면서 자동으로 갱신된다.
- **읽기 장벽(Read Barrier)**
   모든 객체 접근 경로에 얇은 읽기 장벽을 삽입하여,
   객체가 이동 중이거나 이미 복사된 경우 새 주소로 포워딩하도록 한다.
   이를 통해 애플리케이션 스레드는 “항상 최신 객체를 참조한다”는 불변식을 유지할 수 있다.
- **연결 행렬(Connection Matrix)**
   Region 간 참조 관계를 추적하기 위한 자료구조이다.
   G1의 Remembered Set보다 단순하고 메모리 사용량이 적다.

이러한 설계 덕분에 Shenandoah는 GC 중에도 **거의 끊김 없는 실행 흐름**을 제공한다.

------

### ZGC (Z Garbage Collector)

**ZGC**는 Oracle이 개발한 저지연 컬렉터로, JDK 11에서 실험 버전으로 등장하고 JDK 15에서 정식으로 도입되었다.  JDK 21에서는 **세대 구분 모드(Generational ZGC)** 가 추가되었다.

------

#### 주요 특징

- **Region 기반 동적 메모리 레이아웃**
   힙을 **ZPage**라 불리는 가변 크기의 영역으로 관리한다.
   작은 객체는 Small(2MB), 중간 크기는 Medium(32MB), 대형 객체는 Large(가변, 2MB 배수) Region에 저장된다.
   Large Region은 하나의 대형 객체만 포함하며, 재할당 없이 그대로 노화시킬 수 있다.
- **컬러 포인터(Colored Pointer)**
   ZGC의 핵심 기술로, 64비트 주소 상위 4비트를 추가 메타데이터로 사용한다.
   객체의 “마크 상태”, “이동 여부” 등을 포인터 자체에 기록함으로써,
   객체에 접근하지 않고도 GC 상태를 즉시 판단할 수 있다.
- **읽기 장벽(Read Barrier)**
   ZGC는 쓰기 장벽(Write Barrier)을 전혀 사용하지 않는다.
   모든 객체 접근 경로에 읽기 장벽을 삽입하여, 컬러 포인터 상태를 검사하고 필요한 경우 참조를 자동 갱신한다.
   이로 인해 장벽 오버헤드가 최소화된다.

------

#### 동작 과정

ZGC의 GC 사이클은 다음과 같다.

1. **Mark Start (STW)**
    GC Roots를 기반으로 표시를 시작한다.
2. **Concurrent Mark**
    애플리케이션 스레드와 동시에 객체 그래프를 탐색하며, 살아 있는 객체를 컬러 포인터에 표시한다.
3. **Concurrent Relocate Prepare**
    수집 대상 Region을 선정하고, 복사할 객체 목록을 만든다.
    G1의 Collection Set과 달리, ZGC는 매 사이클마다 모든 Region을 검사한다.
4. **Concurrent Relocate**
    선택된 객체를 새 Region으로 복사하고, 원본 객체의 헤더에 포워딩 정보를 기록한다.
    애플리케이션 스레드가 이전 주소를 참조하면, 읽기 장벽이 자동으로 이를 새 주소로 갱신한다.
    이를 **자가 치유(Self-Healing)** 라고 부른다.
5. **Concurrent Remap**
    이전 사이클에서 남은 참조 갱신 작업을 새 사이클의 표시 단계와 통합하여 수행한다.
    이렇게 함으로써 별도의 정지 시간을 추가하지 않고 객체 그래프를 한 번만 스캔한다.

------

### 세대 구분 ZGC (Generational ZGC)

JDK 21에서 추가된 **Generational ZGC**는 기존 ZGC에 세대 구분(Young / Old Generation)을 도입한 버전이다.
 이는 “짧은 수명의 객체를 더 자주 회수”하여 전체 효율을 높이기 위한 개선이다.

------

#### 주요 개선점

1. **이중 버퍼 기반 Remembered Set**
    기존의 포인터 테이블 대신 비트맵을 활용하여 객체 필드 단위의 참조를 추적한다.
    비트 하나가 객체 필드 주소 하나를 의미한다.
2. **밀집도 기반 Region 선택**
    Young Generation의 Region 중 생존 객체 비율(밀집도)이 낮은 영역을 우선 회수한다.
    밀집도가 높다면 해당 Region을 Old Generation으로 승격시킨다.
3. **거대 객체의 직접 할당**
    대형 객체는 처음부터 Old Generation에 배치되며, 복사 없이 노화시킬 수 있다.
4. **다중 매핑 제거**
    이전 ZGC에서 사용하던 3중 가상 메모리 매핑 기법을 제거하고,
    그 공간을 포인터 메타데이터 확장에 활용한다.

## 적합한 가비지 컬렉터 선택하기

적절한 Garbage Collector를 선택하기 위해서는 다음 세 가지 요인을 함께 고려해야 한다.

- **애플리케이션의 주 목적** - 사용자 응답 속도가 중요한지, 배치형으로 처리량이 더 중요한지, 혹은 단순한 테스트·도구용 프로그램인지에 따라 선택이 달라진다.
- **구동 환경(Subsystem)** - 애플리케이션이 단일 프로세서 환경인지, 멀티코어 서버 환경인지, 혹은 클라우드 환경에서 동적으로 리소스를 할당받는지에 따라 GC의 병렬성 및 동시성 효과가 달라진다.
- **JDK 벤더 및 버전** - 일부 GC는 특정 벤더(JDK 버전)에 한정되어 제공 됨 (예를 들어 Shenandoah는 Red Hat 기반 OpenJDK에서 우선 도입되었고,  ZGC는 Oracle OpenJDK 계열에서 정식 지원되었다.)

------

Oracle은 공식적으로 다음과 같은 가이드라인을 제시한다.

| 상황                                                         | 권장 GC                        |
| ------------------------------------------------------------ | ------------------------------ |
| 처리 데이터가 수십~수백 MB 수준으로 작고, 단일 스레드 기반인 경우 | **Serial GC**                  |
| 단일 프로세서 기반이며, 정지 시간이 중요하지 않은 경우       | **Serial GC**                  |
| 전체 처리량(Throughput)이 가장 중요한 경우                   | **Parallel GC** (또는 기본 GC) |
| 응답 시간이 중요하고, STW가 짧아야 하는 경우                 | **G1 GC**                      |
| 초저지연 응답이 요구되는 경우 (예: 실시간 API, 트레이딩 시스템 등) | **Generational ZGC**           |

------

## 가상 머신과 GC 로그

 JDK 9부터는 새로운 `Unified Logging` 시스템이 도입되었으며,
 `-Xlog` 옵션으로 다양한 로그 태그와 출력을 지정할 수 있다.

```
-Xlog[:[selector][:[output][:[decorators][:output-options]]]]
```



 태그(tag)와 로그 레벨(level)을 조합하여 어떤 로그를 얼마나 상세히 출력할지 결정한다.

- **주요 태그(tag)** : `gc`, `heap`, `safepoint`, `age` 등
- **로그 레벨(level)** : `trace`, `debug`, `info`, `warning`, `error`, `off` (6단계)

------

### 대표적인 설정 예시

| 목적                     | 옵션                  | 설명                                                 |
| ------------------------ | --------------------- | ---------------------------------------------------- |
| 기본 GC 로그 출력        | `-Xlog:gc`            | GC 이벤트 요약을 출력한다.                           |
| 상세 GC 로그             | `-Xlog:gc*`           | GC 관련 세부 이벤트를 모두 출력한다.                 |
| 힙·메서드 영역 변동 확인 | `-Xlog:gc+heap=debug` | GC 전후 힙 사용량 및 영역별 변화를 기록한다.         |
| Safepoint 구간 분석      | `-Xlog:safepoint`     | 사용자 스레드 일시 정지 시간과 복귀 시점을 확인한다. |
| 객체 나이 분포 확인      | `-Xlog:gc+age=trace`  | 생존 객체들의 age histogram을 출력한다.              |

------

### 로그 데코레이터 (Decorators)

각 로그에는 추가 정보를 붙일 수 있다.
 `decorators` 항목을 지정하지 않으면 기본적으로 `uptime`, `level`, `tag`가 포함된다.

| 데코레이터     | 의미                                 |
| -------------- | ------------------------------------ |
| `time`         | 현재 날짜 및 시간                    |
| `uptime`       | JVM 시작 이후 경과 시간(초)          |
| `timemillis`   | 현재 시각(밀리초 단위)               |
| `uptimemillis` | JVM 시작 이후 경과 시간(밀리초 단위) |
| `pid`          | 프로세스 ID                          |
| `tid`          | 스레드 ID                            |
| `level`        | 로그 레벨 표시                       |
