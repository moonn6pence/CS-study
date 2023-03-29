# 2주차 - 신현철

## 💡 학습 키워드

- MySQL 엔진 아키텍처
- InnoDB 스토리지 엔진 아키텍처

## 📚 학습 정리

- MySQL 엔진 아키텍처
    - MySQL의 전체 구조
        
        ![MySQL Server Structure](https://user-images.githubusercontent.com/65756225/228421298-bb1474e3-c781-4a33-a65f-14452cbdb1fa.png)
        
        - MySQL 엔진 ↔ Storage 엔진 간에는 Handler API로 명령 주고 받음.
        - MySQL 엔진은 하나 뿐, 스토리지 엔진은 여러 개 둘 수 있음. (테이블마다 다른 스토리지 엔진 사용 가능)
    - MySQL 스레딩 구조
        
        ![MySQL Threading Model](https://user-images.githubusercontent.com/65756225/228421324-c0fff9eb-8bd2-4b1e-bd58-21211d8874ea.png)
        
        - MySQL 서버는 스레드 기반으로 동작한다.
            
            <aside>
            💡 대용량 데이터 처리에 특화된 PostgreSQL의 경우 멀티프로세스 방식임.
            
            [Re: Reasoning behind process instead of thread based](https://www.postgresql.org/message-id/1098894087.31930.62.camel@localhost.localdomain)
            
            </aside>
            
        - MySQL8.0은 멀티 코어 프로세서에서 작동하는가?
            
            <aside>
            💡 Yes. MySQL is fully multithreaded, and makes use of all CPUs made available to it. Not all CPUs may be available; modern operating systems should be able to utilize all underlying CPUs, but also make it possible to restrict a process to a specific CPU or sets of CPUs.
            
            On Windows, there is currently a limit to the number of (logical) processors that **[mysqld](https://dev.mysql.com/doc/refman/8.0/en/mysqld.html)** can use: a single processor group, which is limited to a maximum of 64 logical processors.
            
            Use of multiple cores may be seen in these ways:
            
            - A single core is usually used to service the commands issued from one session.
            - A few background threads make limited use of extra cores; for example, to keep background I/O tasks moving.
            - If the database is I/O-bound (indicated by CPU consumption less than capacity), adding more CPUs is futile. If the database is partitioned into an I/O-bound part and a CPU-bond part, adding CPUs may still be useful.
            
            [MySQL :: MySQL 8.0 Reference Manual :: A.1 MySQL 8.0 FAQ: General](https://dev.mysql.com/doc/refman/8.0/en/faqs-general.html)
            
            </aside>
            
        - 스레드 모델은 2 가지가 있다.
            - 전통적인 스레드 모델
                - 스레드와 커넥션이 1:1 관계를 갖음.
            - Thread Pool 모델
                - 하나의 스레드가 여러 커넥션 요청을 전담
        - Thread는 2 가지로 구분된다.
            - Foreground (클라이언트 스레드)
                - 서버에 접속한 클라이언트만큼 존재한다.
                - 요청 받은 쿼리를 실행한다.
                - 커넥션이 종료되면 스레드 캐시로 돌아가며, 스레드 캐시에 대기 중인 스레드가 꽉차있으면 그냥 종료시킴
                - InnoDB에서는 버퍼, 캐시까지만 처리한다.
            - Background
                - 로그 → 디스크 write
                - 버퍼 풀 → 디스크 write
                - Disk → 버퍼 read
                - Lock, DeadLock 모니터링
                - InnoDB 버퍼→디스크 쓰기 작업 처리
                - Write thread는 갯수 충분히 설정하는게 좋음.
                - Read는 지연 처리 불가능, 하지만 Write은 버퍼링해서 일괄 처리함.
    - 메모리 구조
        - Global Memory
            - 모든 스레드에 공유되는 공유 메모리 영역
            - 테이블 캐시, InnoDB 버퍼 풀, InnoDB 어댑티브 해시 인덱스, InnoDB 리두 로그 버퍼
        - Local Memory (Session Memory)
            - 클라이언트 스레드가 각 쿼리를 처리하는데 사용하는 영역.
            - 스레드에 독립적이라서 공유 안한다.
            - Sort Buffer, Join Buffer, Binary Log Cache, Network Buffer
    - 쿼리 실행 구조
        
        ![Query Execution Structure](https://user-images.githubusercontent.com/65756225/228421332-344ecb63-93ed-461f-9f03-ef460c436ba0.png)
        
        - Parser → Preprocessor → Optimizer → Execution Engine → Storage Engine → Data file
        - Parser : 쿼리를 토크나이징해서 트리로 만들고, 정적으로 에러 검출한다.(신택스 에러 느낌)
        - Preprocessor : 동적으로 테이블, 권한 등과 맵핑까지 해보고 에러가 있는지 검사한다.
        - Optimizer : 쿼리 실행 계획을 짠다.
        - Execution Engine : 쿼리 실행 계획에 따라서 스토리지 엔진에 핸들러 api 호출
    - 스레드 풀
        - EE버전은 내장 기능이지만, 커뮤니티 에디션은 Percona Server 플러그인 사용한다.
        - CPU 코어 수에 맞춰서 processor affinity 높이고, 컨텍스트 스위치 낮출 수 있음.
- InnoDB 스토리지 엔진 아키텍처
    - 주요 특징
        - PK 클러스터링
            - InnoDB에서는 PK 기준으로 클러스터링을 한다.
            - 이 부분은 잘 몰라서 추후에 업데이트 하겠다.
        - FK 지원
            - 스토리지 엔진 레벨에서 지원한다.
            - 외래 키는 데드락 확률을 높인다. → 따라서 끄고 싶으면 `foreign_key_checks`를 `OFF` 설정하자
        - MVCC
            - MVCC를 사용함으로써 잠금 없이 일관성 있는 읽기가 가능해진다.
            - READ_COMMITTED 이상의 격리 수준에서는 트랜잭션 커밋 안된 데이터는 읽지 않아야함 → 버퍼 풀의 값을 바로 읽지 않고, 트랜잭션 중인 데이터는 UNDO 로그의 값을 리턴한다.
        - Non-Locking Consistent Read
            - [1주차 내용 참고](https://www.notion.so/1-836e42b7075b4f9cb887d8a0ca5cc3c1)
        - 자동 데드락 감지
            - Wait-for List 형태로 대기 목록을 관리한다.
            - 주기적으로 이 리스트를 체크해서 데드락 생기면 언두로그가 적은 트랜잭션을 먼저 종료한다.
            - 동시 처리 스레드가 많아지거나 트랜잭션의 잠금 수가 증가할수록 감지 스레드의 성능이 저하된다. 따라서 `innodb_deadlock_detect` 변수로 비활성화 가능하다.
            - 하지만 데드락 해결이 안되므로 `innodb_deadlock_detect` 을 OFF 한다면`innodb_lock_wait_timeout` 시간을 낮추는 것이 바람직하다.
        - 자동화된 장애 복구
            - MySQL 서버가 시작할 때 미완료 트랜잭션이나 partial write된 페이지를 자동 복구한다.
            - InnoDB는 견고해서 데이터 파일 손상이 적지만, 디스크나 하드웨어 이슈로 자동 복구에 문제가 생길 수 있다. → MySQL 서버가 종료되어버린다. → `innodb_force_recovery` 시스템 변수를 설정해서 서버를 시작해야한다.(1부터 6까지 값을 변경해보면서 재시작)
            - 위 방법보다 풀 백업과 바이너리 로그로 복구하는게 손실이 적을 수 있다.
        - InnoDB Buffer Pool
            - 버퍼 풀 전체를 관리하는 세마포어의 경합 때문에 버퍼 풀을 쪼개서 버퍼 풀 인스턴스 여러 개로 관리한다.
            - Page 단위로 메모리 공간을 쪼개서 데이터를 page에 저장하고, 이를 관리하기 위해 3 가지 자료 구조를 사용한다.
            - 버퍼 풀 구조
                - LRU 리스트
                    - LRU + MRU 구조
                    - 새로운 페이지는 MRU의 tail에 추가한다.
                    - LRU의 데이터가 실제로 읽히면 MRU 헤더로 이동한다.
                    - Age가 크면 LRU tail에 위치하게 되고, 결국 버퍼 루에서 제거된다.
                - Flush 리스트
                    - Dirty page를 변경 시점을 기준으로 페이지 목록을 관리한다.
                    - 데이터에 변경이 생겼다면, 언젠가는 디스크와 동기화를 해줘야하고 이 것을 플러시 리스트로 관리한다.
                - Free 리스트
                    - 버퍼 풀에서 실제 사용자 데이터가 아닌 빈 페이지 목록
            - 버퍼 풀과 REDO 로그
                - InnoDB 버퍼 풀의 더티 페이지는 특정 REDO 로그 엔트리가 가리키고 있다.
                - 체크포인트가 발생하면 체크포인트 LSN보다 작은 REDO 로그가 가리키는 더티 페이지 & 리두 로그 엔트리는 모두 디스크로 동기화된다.
                - 리두 로그 파일에서 재사용 불가능한 공간은 Active Redo Log라고 한다.
                - Redo 로그 공간이 무조건 큰게 좋은 게 아니다. 너무 크면 리두 로그가 쌓여서 IO 버스트가 일어난다.
            - 버퍼 풀 Flush
                - 더티 페이지를 디스크에 동기화하는 기능
                - Flush List와 LRU List 라는 두 개의 플러시를 백그라운드 스레드로 실행한다.
                - Flush List Flush
                    - REDO 로그 엔트리 공간은 제한적이므로 이미 동기화가 끝난 로그 공간에는 덮어써서 새로운 로그를 쓴다. → 공간 재활용을 위해서는 더티 페이지가 디스크로 동기화되어야한다. → 주기적으로 Cleaner 스레드가 Flush_list flush 함수를 호출한다.
                    - 고려할 시스템 변수들
                        - 클리너 스레드 갯수
                        - 더티 페이지 최대 비율 : 크면 버퍼링 많이 해서 좋지만 너무 클 경우엔 IO 버스트해짐
                        - Adaptive Flush : 리두 로그 증가 속도를 분석해서 적절한 더티 페이지 비율을 유지하면서 Disk IO 실행
                - LRU List Flush
                    - LRU 리스트에서 사용 빈도가 낮은 페이지를 동기화하면서 제거한다.
                    - 제거된 클린 페이지는 Free List에 들어간다.
                    - 버퍼 풀 인스턴스별로 수행된다.
            - 버퍼 풀 상태 백업 및 복구
                - 버퍼 풀에 데이터가 로드되어 있는 Warming Up 상태면 쿼리 성능이 향상된다.
                    - MySQL 서버가 재시작될 때 워밍업 시키기 위해서 InnoDB 버퍼 풀 상태 백업 기능이 존재한다.
                    - 백업은 메타 데이터만 저장해서 빠르지만, 복구는 실제 디스크에서 읽어오므로 시간이 걸림.
                - 버퍼 풀에 로드된 내용도 확인이 가능하다.
        - Double Write Buffer
            - 리두 로그의 더티 페이지를 플러시할 때 Partial-page를 막기 위한 기능이다.
            - SSD 같은 스토리지는 이 기능이 비효율적임. 따라서 데이터의 무결성이 중요할 때만 쓰자.
            1. 디스크에 동기화 전에 DoubleWrite 버퍼에 먼저 더티 페이지들을 묶어서 한 번에 기록한다.
            2. 각 더티 페이지를 데이터 파일의 적당한 곳에 랜덤으로 쓴다.
            3. 서버가 비정상 종료 되고 재시작될 때, InnoDB 엔진은 DoubleWrite 버퍼를 체크한다.
            4. 데이터 파일과 DoubleWrite 버퍼의 내용이 다르다면 버퍼의 내용을 데이터 파일의 페이지로 복사한다.
        - UNDO 로그
            - DML 실행 이전의 데이터를 백업한 데이터.
            - Undo Tablespace라는 외부 로그 파일에 저장된다.
            - UNDO 로그의 사용
                - 트랜잭션 롤백할 때 사용한다.
                - 격리 수준에 맞게 데이터 반환할 때 사용한다.
            - 트랜잭션이 안 끝나면 언두 로그가 계속 쌓인다.
            - InnoDB가 알아서 관리해준다.
        - Change Buffer
            - INSERT나 UPDATE로 인한 데이터 변경은 인덱스 업데이트가 일어나는데, 이 인덱스 변경 작업은 디스크에 랜덤IO가 필요하다. 따라서 체인지 버퍼에 버퍼링해놓고 나중에 동기화한다.
        - REDO 로그 및 로그 버퍼
            - 리두 로그 및 로그 버퍼
                - 리두 로그는 Durability를 위한 기능임. (커밋된 트랜잭션을 영원히 반영)
                - 디스크에 동기화 안된 데이터를 잃지 않도록 하는 안전장치 역할을 한다.
                - WAL을 따른다.
                - 리두 로그는 write 비용이 낮은 자료 구조를 갖느다.
                - 리두 로그 또한 디스크에 바로 쓰지 않고, 로그 버퍼를 통해서 버퍼링한다.
                - 리두 로그를 버퍼에 write 하는 것과 디스크까지 sync하는 것의 주기를 설정가능하다.
                    
                    ![InnoDB Redo Log Write and Sync](https://user-images.githubusercontent.com/65756225/228421336-3ea9b9e4-e985-4b06-8f25-93bf76099c9f.png)
                    
                - 로그 버퍼 크기는 16MB 정도가 적당함.
            - 리두 로그 아카이빙
                - 리두 로그 복제 하는 속도를 리두 로그가 쌓이는 속도보다 느리면, 덮어써져서 손실이 발생 → 복제 실패
                - 아카이빙 기능을 쓰면 리두 로그 백업을 보장해준다.
            - 리두 로그 활성화 및 비활성화
                - 리두 로그는 데이터 파일과 다르게 커밋되면 항상 디스크까지 기록한다.(설정마다 다르지만, 이렇게 하는게 일반적임) → 대용량 데이터 로드하면 느림
                - 리두 로그 쓰기 속도 때문에 비활성화 가능함. 그러나 일관성이 깨지므로 왠만하면 켜주자.

## ❓ 질문

## 🗂️ 참고자료

[Aurora MySQL vs Aurora PostgreSQL | 우아한형제들 기술블로그](https://techblog.woowahan.com/6550/)

[[MySQL] InnoDB - Redo Log](https://myinfrabox.tistory.com/259)