# 1주차 - 신현철

## 💡 핵심 키워드

- Disk Space Management
- Buffer Management
- 트랜잭션, 트랜잭션 격리 레벨, 2 Phase Lock

## 🔥 집중 키워드

- 트랜잭션
- 트랜잭션 격리 레벨
- 2 Phase Lock

## 📚 학습 정리

- Lock vs Transaction
    - Lock은 동시성을 제어하기 위한 기능
    - Transaction은 데이터의 정합성을 보장하기 위한 기능임.
- InnoDB ?
    
    <aside>
    💡 InnoDB
    
    - MySQL을 위한 데이터베이스 엔진(5.5버전부터 디폴트)
    - 오라클에서 라이선스 소유 중이다.
    - Tablespace라는 개념을 사용한다.
    - 트랜잭션 처리가 필요하고, 대용량 데이터를 다룰 때 효율적이다.
    - PostreSQL을 닮은 ACID 호환 트랜잭션에 대응한다고 한다.
    </aside>
    
- MyISAM 과 InnoDB 처리 방식 차이
    - MyISAM은 트랜잭션 지원 X
    - InnoDB는 트랜잭션 지원 O
    - 예시 상황
        1. PK 3인 밸류가 이미 존재
        2. PK 1,2,3을 한번에 insert하는 상황(auto commit으로)
        3. InnoDB는 모두 rollback, 반면에 MyISAM이나 MEMORY는 1,2는 들어감
- Naver D2 [DBMS는 어떻게 트랜잭션을 관리할까?] 정리
    - 트랜잭션이란?
        - Transaction ?
            - 하나의 논리적 작업 단위를 구성하는 일련의 연산들의 집합
            - 논리적인 작업 셋을 모두 수행하거나 실패했을 때는 원 상태로 복구해야함 → Partial update 방지
            - ACID를 성질로 설명된다.
            - 트랜잭션이 종료 되는 세 가지 상황
                1. 문제 없이 정상 종료
                    - BEGIN→READ→WRITE→READ→…→WRITE→COMMIT
                2. 잘못된 입력, 일관성 제약 조건 위배 혹은 사용자의 요청에 의해 철회
                    - BEGIN→READ→WRITE→READ→…→ABORT
                3. 타임 아웃, 데드락 등 시스템이 감지하여 DBMS가 철회
                    - BEGIN→READ→WRITE→READ→…→SYSTEM ABORTS TRANSACTION
        - ACID ?
            
            <aside>
            💡 ACID
            
            1. Atomicity
                - 트랜잭션의 작업들이 partial update가 없도록 보장하는 능력
                - 작업 도중 하나라도 실패 → 트랜잭션 단위로 묶여 있는 모든 작업을 rollback
            2. Consistency
                - 트랜잭션이 실행을 성공적으로 완료하면 일관성 있는 데이터베이스 상태로 유지하는 것.
                - 데이터베이스가 가지고 있던 제약이 트랜잭션 후에도 똑같이 지켜져야함.
            3. Isolation
                - 트랜잭션 수행 시 다른 트랜잭션의 연산이 끼어들지 못하게 보장하는 것
                - 독립적인 여러 트랜잭션을 동시에 수행한다고 해도, 중첩된 상태가 아닌 순차적으로 실행되는 것과 동일한 결과가 나타나야함.
                - 동시 실행과 연속 실행의 결과가 같아야함.
            4. Durability
                - 트랜잭션 수행 성공 시 영원히 반영되어야함을 의미 = 로그가 남아야함
            - ACID 구현은 매우 어렵다고 함
                - 추가로 공부할 것 : 로깅 방식, shadow paging, MVCC
            </aside>
            
    - 트랜잭션 관리를 위한 DBMS의 전략
        - DBMS의 개략적인 구조
            - 비휘발성 저장 장치인 디스크에 데이터 저장
            - DB의 일부분을 MM에 유지
            - 데이터의 고정 길이를 page로 저장 → 디스크에서 read, write할 때 page 단위로 입출력이 수행된다.
            - 보통 Query Processor + Storage System으로 구성된다.
            - MySQL은 상부 쿼리 프로세서, 하부 스토리지 시스템으로 나뉘어져 있는 layered 구조임
            - InnoDB 같은 엔진이 하부 스토리지 시스템임.
            - 스토리지 시스템 안에 페이지 버퍼가 존재함.
        - 버퍼 관리자
            - MM에 유지하는 페이지들을 관리하는 모듈
            - Page Buffer라고도 한다.
        - UNDO와 REDO
            
            <aside>
            💡 UNDO
            
            - 롤백, 읽기 일관성, 복구 역할
            - 사용자가 했던 작업을 반대로 진행해서 원상태로 돌린다.
            </aside>
            
            <aside>
            💡 REDO
            
            - 무슨 작업을 하던 REDO에 기록됨
            - 사용자가 했던 작업을 그대로 진행해서 복구한다.
            </aside>
            
            - 시스템 장애 발생 → UNDO 데이터도 날아감 → REDO 데이터를 이용해서 마지막 check point부터 장애까지의 DB Buffer Cache를 복구함 → 최종적으로 UNDO를 이용해 commit안된 데이터 모두 rollback
            - 둘 다 Idempotent 성질을 가져야함.
        - FORCE와 STEAL
            
            <aside>
            💡 수정된 페이지를 트랜잭션 종료 시점에 디스크 쓸 것인가?
              FORCE : 바로 기록한다.
            ¬FORCE : 바로 기록하지 않는다. (REDO 필요함)
            
            </aside>
            
            <aside>
            💡 트랜잭션이 완료되지 않은 상태에서 데이터를 디스크에 기록할 것인가?
            (수정된 페이지를 디스크에 쓰는 시점에 관련된 정책)
              STEAL : 수정된 페이지를 언제든지 디스크에 쓴다. (UNDO 필요함)
            ¬STEAL : 수정된 페이지를 최소한 EOT 까지는 버퍼에 유지한다.
            
            </aside>
            
            - EOT : End of Transaction
            - 트랜잭션이 완료되지 않더라도 성능 효율 등의 이유로, 대부분의 DBMS에서 적당히 미리 기록하는 메커니즘을 사용한다.
        - 트랜잭션 관리와 연관된 버퍼 관리 정책
            - 버퍼 관리 정책이 트랜잭션 관리에 밀접하게 연관되어 있다 → 버퍼 관리 정책에 따라 트랜잭션의 UNDO 복구과 REDO 복구가 요구되거나 그렇지 않게 된다.
            - UNDO가 필요한 이유
                - 트랜잭션이 완료되지 않더라도 오퍼레이션 도중에 수정된 page 들이 디스크에 쓰여질 수 있음
                - 따라서 트랜잭션이 비정상 종료된다면 수정된 내용들은 UNDO를 통해 원상 복구해야함
                - 버퍼 관리자가 트랜잭션 종료 전에 수정된 Page들을 디스크에 쓰지 않으면 UNDO는 메모리 버퍼에서만 이루어지면 되어서 간단함 → But, 메모리 버퍼가 많이 필요함.
            - REDO가 필요한 이유
                - FORCE 정책은 REDO가 필요 없지만, DB 백업으로부터 복구 과정에서는 필요함.
                - No-FORCE 정책을 채택하면 수정된 페이지가 디스크에 반영하지 않는다해도, 커밋 시점에 로그는 남김. 커밋 내역이 디스크에 반영 안되어 있을 수 있으므로 반드시 REDO가 필요함.
            - 대부분의 DBMS에서 STEAL & ¬FORCE를 사용한다. → UNDO와 REDO가 필요함
        - Log ?
            
            <aside>
            💡 Log : UNDO 복구과 REDO 복구를 위해 가장 많이 쓰이는 구조. Log Record의 연속. Stable storage에 기록된다. 하지만 이론적으로는 이상적인 stable storage는 없음. (RAID 같은 인프라 시스템을 통해서 멀티 로그를 만들어 유사 stable storage를 구현은 가능하다.) 성능 상의 이유로 보통 하나의 로그를 유지
            
            </aside>
            
        - RAID ?
            
            <aside>
            💡 Redundant Array of Independent Disks
            
            </aside>
            
            - 여러 하드 디스크에 일부 중복된 데이터를 나눠서 저장하는 기술
            - 데이터를 나누는 방법인 Level에 따라서 목적이 다름
        - Shadow Paging ?
            
            <aside>
            💡 Shadow Paging : 하드디스크에 트랜잭션이 시작 시점에 current page table의 복제본인 shadow page table을 생성한다.
            트랜잭션 정상 종료 시에는 shadow page table 삭제함.
            트랜잭션 실패 시에는 shadow page table을 current page table로 사용함.
            
            </aside>
            
        - 트랜잭션 관리
            - 로그
                - append 방식으로 기록되며 각 로그 레코드가 고유 id(LSN or LSA) 갖음.
                - append 방식이라 id도 단조증가함.
                - 기록하는 오브젝트 타입에 따라 Logical과 Physical로 분류 + State or Transition으로 분류
                
                |  | State | Transition |
                | --- | --- | --- |
                | Logical |  | DML, DDL |
                | Physical | 이전 이미지
                이후 이미지 | XOR 차이 |
                - 로깅 방법들
                    - Physical State Logging
                        - 가장 널리 쓰이는 기본적인 로깅 방법임.
                        - 로그 레코드는 갱신 이전 이미지와 이후 이미지를 모두 다 가지고 있음.
                        - UNDO 때는 이전 이미지로 현재 이미지를 대체한다.
                        - REDO 때는 이후 이미지를 반영한다.
                        - 페이지 레벨, 레코드 레벨에서 이루어진다.
                    - Physical Transition Logging
                        - 이전 이후 이미지를 갖고 있는게 아닌, XOR 차이점을 기록한다.
                    - Logical Transition Logging (Operation Logging)
                        - 어떤 일을 했었는지 기록한다.
                        - REDO 시에는 해당 operation을 수행
                        - UNDO 시에는 역 operation을 수행
                        - 로그 레코드 크기를 크게 줄여준다.
                        - 물리적으로 복구하기 어려운 자료 구조에 대한 로깅을 쉽게 해줌.
                        - 인덱스 구조에 사용되는 자료구조에 따라서 레코드의 위치가 계속 변경됨 → 로깅 시점과 복구 시점의 데이터의 물리적 위치가 달라짐 → 물리적인 로그보다는 논리적인 로그로 복구하는게 수월함.
                    - Physiological Logging
                        - 수정한 페이지를 물리적으로 식별 + 해당 페이지 내에서는 논리적으로 기록됨
                - DBMS는 여러 로깅 방법들을 적절하게 섞어 씀.
            - 로그는 어떻게 쓸까?
                - 무조건 두 가지 규칙을 따름
                    - WAL : DB에 update 치기 전에 UNDO 정보가 로그에 써져야 한다.
                    - 트랜잭션이 정상 종료 처리되기 위해서는 먼저 REDO 정보가 로그에 쓰여져야함. (적어도 커밋 시점에)
                - 로그 버퍼에 로그 레코드를 모았다가 블록 다위로 로그 파일에 출력
                - 트랜잭션들이 수행되면서 로그 레코드가 생기고 로그 버퍼에 들어감
                    
                    → 로그 버퍼에서 특정 시점에 로그 파일에 써짐
                    
                - 로그가 버퍼 → 파일에 출력되는 시점
                    - 트랜잭션이 커밋 요청
                    - WAL 해야하는 경우
                    - 로그 버퍼 모두 소진 (긴 트랜잭션에서 발생 가능)
                    - DBMS가 checkpoint 연산, 로그 관리 연산 등으로 필요로 하는 경우
            - 로그를 쓰는 일은 왜 느릴까?
                - 로그 레코드의 손실은 완벽한 데이터베이스 복구 불가능으로 이어진다 → DBMS에서는 로그 레코드를 안전하게 쓰기 위해 write, fsync 같은 함수를 호출 → fsync는 매우 느리지만 트랜잭션은 이 함수 호출이 종료되길 기다려야함.
                - 로그 버퍼 쓰는 과정 : [로그 레코드 wirte & fsync → 로그 헤더 write & fsync] 반복
                - 로그 파일 쓰는 과정 : [여러 로그 페이지를 한 번에 write & sync] 반복
                - 대부분의 커밋 연산이 소모하는 시간은 로그 레코드를 로그 파일에 쓰고, fsync 함수를 실행하는 시간임.
            - 커밋 오퍼레이션 성능 개선
                - Group Commit
                    - 트랜잭션의 커밋을 그룹 단위로 처리 → response time은 느려지지만, 전체적인 throughput 향상됨.
                    - 동시 요청되는 커밋이 많을수록 효율 증가
                    - 그룹 커밋 범위 설정은 성능에 매우 중요함(n ~ nnn ms 까지 다양하게 설정)
            - 성능을 위해서 지속성을 살짝 포기할 수는 없을까?
                - 커밋 성능을 위해 durability 일부 포기하는 방식도 있다.
                    - DBMS에서 매 커밋마다 정확하게 로그 write & fsync가 아닌 느슨한 옵션 제공
                    - InnoDB : innodb_flush_log_at_trx_commit 파라미터 설정
                - 비동기 커밋 방식을 사용하기도 함.
                    - 로그 레코드 써질 때까지 대기 안하고 커밋.
                    - 로그 쓰기 thread나 process에서 로그 레코드 write.
                    - 장애로 인해서 커밋은 되었지만, 로그는 안써지는 loss 발생 가능.
                        - DBMS에서 커밋이 안된 것으로 간주하고 rollback으로 처리함.
                        - Rollback을 위한 UNDO 로그는 이미 WAL원칙에 의해 트랜잭션 로그에 써져 있음.
                    - 성능 향상은 좋지만, loss 발생 가능. → 동시 요청 많으면서, 일부 유실이 있어도 괜찮은 환경에 적합함.
            - 어떻게 로그로 복구가 이루어지나?
                - 복구 종류
                    - 트랜잭션 철회는 어떻게?
                        - 로그 역방향 탐색하여 UNDO가 필요한 로그 찾아서 트랜잭션 수행 순서의 역순으로 UNDO 수행
                        - UNDO 이후에는 보상 로그 레코드(CLR)라는 REDO 전용 로그를 씀.
                            - UNDO 재발생 방지
                    - 장애로 인해 재시작되면 어떻게 복구가 되나?
                        
                        <aside>
                        💡 3단계 복구
                        
                        1. 로그 분석 단계
                            - 마지막 체크포인트 ~ EOL까지 로그 탐색하여 어디서부터 시스템이 복구해야하는지 알아내는 단계
                        2. REDO 복구 단계 (Reapting History)
                            - 복구 시작 시점 ~ 장애 발생 직전까지 REDO가 필요한 모든 로그를 REDO 방식으로 복구
                            - 장애 발생 시점과 같은 상태로 돌아감.
                        3. UNDO 복구 단계(Global UNDO)
                            - 최신 시점부터 역방향으로 탐색하면서 UNDO가 필요한 로그들에 대해 UNDO 복구
                            - 여러 트랜잭션을 철회한다는 점이 ‘트랜잭션 롤백’과의 차이임.
                        </aside>
                        
                        - UNDO나 REDO가 필요한지 판단 방법 : ARIES recovery 알고리즘
                            - 로그의 LSN과 page LSN 비교
                            - 모든 페이지는 Page LSN을 갖음. Page LSN은 페이지가 갱신될 때마다 해당 로그의 LSN으로 갱신됨.
                            - Page LSN이 임의의 로그의 LSN 보다 오래됨 → 해당 페이지는 해당 로그로 복구되어야함.
                - 백업을 이용한 미디어 복구는 어떻게?
                    - 미디어 복구 → Archive 복구 : DB의 백업으로부터 복구하는 것
                        - 백업 기법 → fuzzy : snapshot을 그대로 복사. 미처 커밋 못한 일부 트랜잭션의 이미지가 복사되거나 커밋된 것이 아직 반영 안될 수도 있음 → 로그로 해결!
                        - 장애 발생 시 복구 문제와 같은 원리이다.
                        - fuzzy하게 복사한 DB 이미지에서 미처 반영되지 못한 커밋했던 트랜잭션들을 REDO해주고, 커밋 레코드가 포함되지 않은 트랜잭션은 UNDO해준다.(roll-forward 복구)
            
        - 커밋을 하면 어떤 일이 일어나나?
            - 커밋을 하면 어떤 일이?
                - Commit trigger나 deferred trigger가 정의되어 있다면 해당 트리거가 수행된다.
                - 트랜잭션 수행 도중에 생성했던 커서를 정리한다.
                    - Cursor : 질의 결과 집합
                - SQL 표준은 Holdable Cursor만 유지하고 나머지는 해제하는 것.
                - JDBC는 Holdable Cursor가 기본이다. (중첩 트랜잭션의 편의성을 위해서)
            - 커밋을 하다가 오류가 발생하면?
                - 트랜잭션이 커밋이 완료된 것이 아니라면 수행되지 않은 것과 같음.
- Transaction Isolation Level
    - 트랜잭션과 동시성을 모두 얻기 위한 방법
    - 동시에 DB에 접근할 때 어떻게 제어할 것인지에 대한 설정
    - 4가지 격리 수준 존재
        
        ![Untitled](1%E1%84%8C%E1%85%AE%E1%84%8E%E1%85%A1%20-%20%E1%84%89%E1%85%B5%E1%86%AB%E1%84%92%E1%85%A7%E1%86%AB%E1%84%8E%E1%85%A5%E1%86%AF%20836e42b7075b4f9cb887d8a0ca5cc3c1/Untitled.png)
        
        - READ UNCOMMITTED
            - 커밋 전의 트랜잭션의 변경사항이 다른 트랜잭션이 읽는 것을 허용
            - Dirty Read 발생 가능
                
                <aside>
                💡 Dirty Read?
                
                ![Untitled](1%E1%84%8C%E1%85%AE%E1%84%8E%E1%85%A1%20-%20%E1%84%89%E1%85%B5%E1%86%AB%E1%84%92%E1%85%A7%E1%86%AB%E1%84%8E%E1%85%A5%E1%86%AF%20836e42b7075b4f9cb887d8a0ca5cc3c1/Untitled%201.png)
                
                - 데이터 부정합 현상
                - Tx A가 롤백되면 Tx B 입장에서는 무효가 된 데이터를 처리하게 된다.
                </aside>
                
        - READ COMMITTED
            - 커밋이 완료된 트랜잭션의 변경사항만 다른 트랜잭션에서 조회 가능.
            - Record 락만 설정, gap lock은 설정하지 않음.
            - 데드락 확률 높아짐.
            - 각 select 쿼리마다 스냅샷을 생성함.
            - Non-Repeatable Read와 Phantom Read 발생 가능함.
            
            <aside>
            💡 Non-Repeatable Read
            
            - 커밋 되기 전엔 트랜잭션 이전의 값을 가져오고, 커밋이 되면 반영된 값을 가져온다.
            
            ![Untitled](1%E1%84%8C%E1%85%AE%E1%84%8E%E1%85%A1%20-%20%E1%84%89%E1%85%B5%E1%86%AB%E1%84%92%E1%85%A7%E1%86%AB%E1%84%8E%E1%85%A5%E1%86%AF%20836e42b7075b4f9cb887d8a0ca5cc3c1/Untitled%202.png)
            
            - Tx A가 commit 되기 전에는 이전 값을, commit 되면 트랜잭션 반영된 값을 읽음
            - Tx B 입장에서는 한 트랜잭션에서 여러 번 select 하는데 값이 달라짐.
            </aside>
            
        - REPEATABLE READ
            - 커밋이 완료된 트랜잭션의 변경사항만 다른 트랜잭션에서 조회 가능 + 트랜잭션 범위 내에서 조회한 내용이 항상 동일함을 보장함.
            - select 쿼리가 수행된 시점을 기준으로 스냅샷이 생성됨. 이후 다른 트랜잭션에 의해 해당 데이터가 변경되어도 UNDO 로그에 저장된 내용을 기반으로 기존 데이터를 재구성함.
            - Non-Repeatable Read는 발생 안함.
            - Phantom Read 발생 가능
            
            <aside>
            💡 Phantom Read
            
            - 한 트랜잭션 안에서 일정 범위의 레코드를 두 번 이상 읽을 때, 첫 쿼리에선 없던 레코드가 두 번째 쿼리에서 나타나는 현상
            - 다른 트랜잭션이 추가한 레코드에 UPDATE 쿼리를 수행하게 될 경우 발생 가능.
            </aside>
            
        - SERIALIZABLE
            - 커밋안 된 트랜잭션의 데이터를 다른 트랜잭션에서 접근 불가능.
            - 성능 안 좋음.
            - Phantom read 방지 가능
- InnoDB의 트랜잭션 모델
    - InnoDB의 Locking
        - 락의 적용 요소에 따른 분류
            - Shared Lock(s-lock)
                - Row-level lock
                - SELECT를 위한 read lock
                - Shared lock이 걸리면 해당 row는 다른 트랜잭션에 의해 s-lock은 걸릴 수 있지만, x-lock은 불가능
            - Exclusive Lock(x-lock)
                - Row-level lock
                - UPDATE, DELETE을 위한 write lock
                - x-lock이 걸린 row는 다른 트랜잭션에 의한 s-lock, x-lock 모두 불가능
            - Intention Lock
                - Table-level lock
                - 테이블의 특정 row에 row-level 락을 걸기 전에 미리 걸어두는 락
                
                ```sql
                SELECT ... LOCK IN SHARED MODE; // intention shared lock (IS) 가 걸림
                // 이 후 row-level에 s-lock 걸림
                ```
                
                ```sql
                SELECT ... FOR UPDATE;
                SELECT ... FOR INSERT;
                SELECT ... FOR DELETE;
                // intention exclusive lock (IX) 가 걸림
                // 이 후 row-level에 x-lock 걸림
                ```
                
                - IS, IX 락은 여러 트랜잭션이 동시 접근 가능.
                - 같은 row에 동시 접근하는 row-level 락을 제어를 하는 역할.
                - LOCK TABLES, ALTER TABLE, DROP TABLE이 실행될 때는 IS, IX가 모두 table-lock으로 블락된다. = IS, IX를 획득하려는 트랜잭션이 대기상태로 바뀜
                - IS, IX 락이 이미 걸려있는 테이블에 LOCK TABLES, ALTER TABLE, DROP TABLE을 걸려고 하면 해당 트랜잭션이 대기상태로 빠짐
                - row & table 형태로 2-phase lock으로 lock이 적용되는 이유
                    - Table 락과 row 락이 중복되는 것(스키마 변경)을 방지할 수 있음.
        - 락의 적용 상황에 따른 분류
            - 대표적으로 트랜잭션 격리 수준 구현을 위해서는 3가지 lock이 있다.
            - Record Lock
                - 인덱스 레코드에 거는 락
                - 테이블의 인덱스가 없더라도 InnoDB에서 숨겨진 Clustered Index를 생성하기 때문에 레코드 락 가능
            - Gap Lock
                - 인덱스 레코드 간의 Gap or 첫 번째나 마지막 인덱스 전후의 Gap에 설정되는 락
                - 삽입하려는 곳의 값 존재 여부와 관계 없이 다른 트랜잭션에서 데이터 삽입 불가능
                - PK 뿐만 아니라 Secondary Index에도 동일하게 사용된다.
                - 같은 SELECT 쿼리를 두 번 실행할 때, 다른 트랜잭션에 의해 데이터가 수정되어도 같은 결과가 리턴되는 것을 보장할 수 있다. (Phantom Read 방지)
                
                ```sql
                // 예시
                SELECT name FROM test WHERE id >= 1 FOR UPDATE;
                ```
                
            - Next-Key Lock
                - Record Lock과 해당 인덱스 앞의 Gap에 대한 Gap Lock이 조합된 락
                - 음의 무한대, 양의 무한대 인덱스를 허위 인덱스인 supremum pseudo-record를 통해 설정.
    - Consistent Non-Locking Reads
        - InnoDB에서 동시성 성능을 최대화하기 위해 MVCC(Multiversion Concurrency Control)를 사용한다. MySQL에서는 Consistent Non-Locking Reads라고 부른다.
        - Snapshot 정보를 바탕으로 Locking이 필요하지 않은 Consistent Read를 제공함.
    - Locking Reads
        - Select 쿼리로 조회한 데이터는 기본적으로 Non-Locking Read이기 때문에 다른 트랜잭션에 의해 변경될 가능성이 높다. → 이를 위해 InnoDB는 두 가지 Locking Read를 제공함.
        - SELECT … FOR SHARE
            - 읽은 row에 shared lock 설정함.
            - 다른 트랜잭션은 해당 row를 읽을 순 있지만, shared lock을 설정한 트랜잭션이 커밋되기 전까지는 수정은 불가능함.
            - 아직 커밋되지 않은 다른 트랜잭션에 의해 해당 row가 변경될 경우, 트랜잭션이 종료된 후에 쿼리가 수행된다.
        - SELECT … FOR UPDATE
            - IX 락
            - 조회 시 발견한 인덱스 레코드에 대해 UPDATE 쿼리처럼 row 및 관련 인덱스에 Lock을 설정함.
- 2 Phase Lock
    
    ![Untitled](1%E1%84%8C%E1%85%AE%E1%84%8E%E1%85%A1%20-%20%E1%84%89%E1%85%B5%E1%86%AB%E1%84%92%E1%85%A7%E1%86%AB%E1%84%8E%E1%85%A5%E1%86%AF%20836e42b7075b4f9cb887d8a0ca5cc3c1/Untitled%203.png)
    
    - 트랜잭션들의 직렬화 보장을 위해서 동시성을 제어하는 기법
    - 데드락은 방지할 수 없다.
    - 세 단계로 실행된다.
        1. Growing phase
            - 트랜잭션이 실행되기 시작할 때 필요한 락에 대한 권한 요청
        2. Locked phase
            - 트랜잭션이 모든 락 권한을 얻는 부분. 트랜잭션이 처음 락 해제할 때 3단계로 넘어감.
        3. Shrinking phase
            - 새로운 락 요청 불가능. 획득했던 락 해제만 가능

## ❓ 질문

## 🗂️ 참고자료

[InnoDB](https://ko.wikipedia.org/wiki/InnoDB#cite_note-1)

[ACID](https://ko.wikipedia.org/wiki/ACID)

[[데이터베이스] 트랜잭션의 ACID 성질 - 하나몬](https://hanamon.kr/데이터베이스-트랜잭션의-acid-성질/)

[[DB] 트랜잭션, REDO와 UNDO 개념](https://brownbears.tistory.com/181)

[IT위키](https://itwiki.kr/w/그림자_페이징_회복_기법)

[RAID](https://ko.wikipedia.org/wiki/RAID)

[데이터베이스 버퍼매니저(BUFFER MANAGER)란 무엇인가? | 개발자 이동욱](https://dongwooklee96.github.io/post/2021/12/29/데이터베이스-버퍼매니저buffer-manager란-무엇인가.html)

[[DB] 트랜잭션 : Log](https://inor.tistory.com/16)

네이버 글 요약

[동기식 입출력 함수 - sync(), fsync(), fdatasync()](https://blog.naver.com/PostView.nhn?isHttpsRedirect=true&blogId=neakoo35&logNo=30131080582)

[DBMS :: Transaction (트랜잭션)](http://diaryofgreen.tistory.com/36)

[[DB] ARIES](https://bubble-dev.tistory.com/entry/DB-ARIES)

[데이터베이스 커서](https://ko.wikipedia.org/wiki/데이터베이스_커서)

[트랜잭션 격리 수준](https://tecoble.techcourse.co.kr/post/2022-11-07-mysql-isolation/)

[TWO-PHASE-LOCK(2PL)이란 (draft) | 개발자 이동욱](https://dongwooklee96.github.io/post/2021/04/07/two-phase-lock2pl이란-draft.html)

[[10분 테코톡] 🌼 예지니어스의 트랜잭션](https://www.youtube.com/watch?v=e9PC0sroCzc)

[프로그래밍 초식 : DB 트랜잭션 조금 이해하기 01](https://www.youtube.com/watch?v=urpF7jwVNWs)

[MySQL-InnoDB에서 데드락 정보 확인하기](https://dataonair.or.kr/db-tech-reference/d-lounge/expert-column/?mod=document&uid=52944)

[MySQL InnoDB lock & deadlock 이해하기](https://www.letmecompile.com/mysql-innodb-lock-deadlock/)

[트랜잭션 수준 읽기 일관성 - [종료]구루비 DB 스터디 - 개발자, DBA가 함께 만들어가는 구루비 지식창고!](http://wiki.gurubee.net/pages/viewpage.action?pageId=21200923)