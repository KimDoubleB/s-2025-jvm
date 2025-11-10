# 대규모 JVM 환경에서의 문제 사례 및 해결 전략

## 대용량 힙 환경에서의 GC 지연 문제

### 시스템 환경

- Quad-Core Xeon (64bit), 16GB RAM, CentOS 5.4
- Resin WAS, JDK 5
- JVM 옵션: `-Xms12g -Xmx12g` (고정 12GB Heap)

### 현상

- 웹 사이트가 장시간 응답하지 않는 문제가 빈번하게 발생
- Full GC 시 최대 **14초 정지(Stop-The-World)** 발생

### 원인

- GC 타입: **Parallel Scavenge + Parallel Old**
  - Throughput 중심 → STW 길어짐
- 힙 크기를 과도하게 크게 설정 (12GB)
  - Old 영역에서 거대 객체가 다수 생성, 회수 비용 증가
  - 대용량 Full GC → STW 시간 폭증

### 해결 방법

- JVM 힙을 **1.5~2GB 수준으로 축소**
- 대형 프로세스 1개 → **소형 JVM 여러 개로 Scale-out**
  - 예) `2GB JVM × 5개 = 총 10GB`
- GC를 CMS로 변경하여 STW 감소

### 관찰 및 교훈

- “메모리를 크게 잡으면 성능이 좋아진다”는 오해 → **큰 힙 = 긴 STW**
- 단일 대형 JVM보다 **작은 JVM 다중 구성(Scale-out)** 이 안정적

------

## 클러스터 캐시 동기화로 인한 메모리 Overflow

### 환경

- Dual CPU, 8GB RAM × 2
- WebLogic 9.2, 총 6노드 클러스터
- Global Cache: **JBossCache**

### 현상

- 메모리 Overflow 발생

### 원인

- `+HeapDumpOnOutOfMemoryError` 분석 결과
   → 수많은 `NAKACK` 객체 존재
- JBossCache가 모든 노드의 데이터 수신 여부 확인
   → 재전송·검증 과부하
- 클러스터 트래픽을 처리하지 못하며 메모리 고갈

### 해결 방법

- 글로벌 캐시 제거 → **로컬 캐시 기반 구조로 전환**

### 교훈

- 캐시 공유는 단순히 “성능 향상”이 아니다
- **분산 캐시 전략 부적절 시 메모리·네트워크 병목 유발**

------

## Direct Memory 사용으로 인한 메모리 부족

### 환경

- Core i5, 4GB RAM, Windows 32bit
- CometD 1.1.1, Jetty 7 (NIO 기반 서버 Push)

### 현상

- OOM 발생
- Eden/Survivor는 정상 → **Java Heap 외부에서 문제 발생**

### 원인

- 32bit Windows 주소 공간 한계 (~2GB)
- Direct Memory 사용 (`ByteBuffer`, NIO)
- Direct 영역 + Heap + Thread Stack + Socket Buffer가 OS 제한 초과

### 해결 방법

- Direct Memory 제한 설정

  ```
  -XX:MaxDirectMemorySize
  ```

- Heap + Non-Heap 총합이 OS 한도를 넘지 않도록 설계

- 필요 시 OS 및 JVM 64bit 전환

### 핵심 포인트

- GC가 관리하지 않는 메모리(Direct/NIO)는 별도 관리 필요

------

## 외부 Shell 호출로 인한 성능 지연

### 환경

- Solaris 10, 4 CPU
- GlassFish

### 현상

- CPU 사용률은 활발하나 실제 WAS는 느린 상태

### 원인

- 요청 처리 과정에서 **외부 Shell Script 실행**
- UNIX `fork()` 비용 → 파일 테이블·메모리 Copy overhead

### 해결

- Shell 호출 제거
- Java API 기반 시스템 정보 조회로 변경

------

##  JVM 프로세스 비정상 종료

### 환경

- Dual CPU, 8GB × 2대, 6노드 클러스터

### 현상

- JVM 프로세스 비정상 종료
- `Connection reset` 오류 지속 발생

### 원인

- 외부 시스템 비동기 호출
- 응답 대기 Thread 누적 → Socket 리소스 고갈

### 해결

- 비동기 호출을 **메시지 큐(Message Queue)** 기반으로 변경
   → 백프레셔(back-pressure) 처리

------

## 비효율적 데이터 구조로 인한 GC 부하

### 환경

- 64bit JVM
- `-Xms4g -Xmx8g -Xmn1g`
- Parallel + CMS

### 현상

- 정상 시 Minor GC ~30ms
- Map 객체 100만 개 생성 시 GC 비용 급증

### 원인

- `HashMap<Long, Long>` 사용
- 객체 헤더/포인터 오버헤드 매우 큼
   → 실 데이터 대비 메모리 비효율

### 해결

**GC 튜닝**

```
-XX:SurvivorRatio=65546
-XX:MaxTenuringThreshold=0
-XX:+AlwaysTenure
```

**코드 레벨 개선**

- primitive 기반 자료구조 고려 (`Long` → `long`, `TLongHashMap` 등)
- 자료구조 선택은 GC 성능에 직접적 영향을 준다.

------

## Windows 가상 메모리로 인한 장시간 STW

### 현상

- 프로그램 최소화 시 **1분간 정지**

### 원인

- Windows minimized window → Working Set을 Disk로 Swap
- GC 시 Swap-in 비용 폭발

### 해결

```
-Dsun.awt.keepWorkingSetOnMinimize=true
```

------

## Safepoint 지연으로 인한 STW 증가

### 환경

- HBase cluster / JDK 8 / G1 GC
- Offline 분석 처리(지연 허용)

### 원인

- 특정 스레드가 Safepoint에 늦게 도달 → VMOP Spin 증가

### 분석 옵션

```
-XX:+PrintSafepointStatistics
-XX:+PrintSafepointStatisticsCount=1
```

### 해결

- Safepoint Timeout 활성화

  ```
  -XX:+SafepointTimeout
  -XX:SafepointTimeoutDelay=2000
  ```

- Counted loop tuning (권장하지 않음)

------

## Eclipse 빌드 성능 최적화 예시

### 핵심 개선 전략

- JDK 업그레이드 → 최신 JVM 최적화 이점

- 클래스 검증 비활성화

  ```
  -Xverify:none
  ```

- GC 로그 분석

  ```
  -Xlog:gc*,gc:gc.log
  jstat -gccause <pid>
  ```

- Explicit GC 무효화

  ```
  -XX:+DisableExplicitGC
  ```

------

## 
