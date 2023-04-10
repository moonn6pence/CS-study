# 3주차 - 신현철

## 💡 학습 키워드

- 트랜잭션과 잠금
- 인덱스

## 📚 학습 정리

- Online DDL?
    - 테이블 스키마 변경 도중에도 DML이 실행할 수 있도록하는 기능
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
            - 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 대 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다.
            - 데드락이나 트랜잭션 대기를 유발하므로 줄이는 것이 좋다.
- Transaction Isolation Level
    - 트랜잭션과 동시성을 모두 얻기 위한 방법
    - 동시에 DB에 접근할 때 어떻게 제어할 것인지에 대한 설정
    - 4가지 격리 수준 존재
        
        ![InnoDB Locks](https://user-images.githubusercontent.com/65756225/230903237-d8ff0c62-c572-4841-a715-adc7e867fc51.png)
        
        - READ UNCOMMITTED
            - 커밋 전의 트랜잭션의 변경사항이 다른 트랜잭션이 읽는 것을 허용
            - Dirty Read 발생 가능
                
                <aside>
                💡 Dirty Read?
                
                ![Dirty Read](https://user-images.githubusercontent.com/65756225/230903230-25b8dff0-74cf-4ec3-8e2a-443b09647063.png)
                
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
            
            ![Non-Repeatable Read](https://user-images.githubusercontent.com/65756225/230904394-721d2f72-260f-464a-bf91-a7b6570a73f9.png)
            
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
            
            - InnoDB의 REPEATABLE READ는 Phantom Read이 발생하지 않는다.
        - SERIALIZABLE
            - 커밋안 된 트랜잭션의 데이터를 다른 트랜잭션에서 접근 불가능.
            - 성능 안 좋음.
            - Phantom read 방지 가능
- Index 정리
    - 인덱스란?
        - 테이블의 칼럼 또는 칼럼들의 값과 레코드의 주소를 key-value 쌍으로 저장해둔 것
        - 랜덤 IO 줄여서 성능 향상 유도
    - 인덱스 장점
        - 조회가 빠르다.
    - 인덱스 단점
        - 삽입, 삭제, 수정은 오래 걸린다.
    - 인덱스 종류
        - 역할별 분류
            - PK 인덱스
                - PK를 인덱스로 만들고, PK 특성 그래도 `NOT NULL`, `UNIQUE` 가 적용된다.
                - 자동 생성된다.
            - 세컨더리 인덱스
                - PK를 제외한 나머지 모든 인덱스
        - 알고리즘별 분류
            - B-Tree 인덱스
            - Hash 인덱스
                - 메모리 기반 DB에서 많이 사용한다.
                - 범위 검색, 일부 검색 등은 불가능하다.
        - 중복 여부 분류
            - Unique 인덱스
            - Non-Unique 인덱스
    - B-Tree 인덱스
        - 구조
            - 루트-브랜치-리프 노드로 구성된다.
            - 루트는 브랜치 페이지를, 브랜치는 리프 노드의 페이지 번호를 갖고 있다.
            - 리프 노드는 PK 값을 갖고 있다.
            - InnoDB는 클러스터링 인덱스 구조를 사용하므로 데이터 파일 또한 B-Tree 구조를 갖는다.
        - 키 추가
            - 키 저장할 공간을 먼저 탐색 → 리프 노드 꽉 찼다면 리프 노드를 분리 → 키 추가
            - 쓰기 비용이 크다.
            - 체인지 버퍼를 통해 지연 처리 가능함.
        - 키 삭제
            - 리프 노드에 삭제 마킹
            - 버퍼링 가능
        - 키 변경
            - 키 삭제 → 키 추가
            - 지연처리 가능
        - 키 검색
            - 컬럼의 일부 값만 검색할 때는 Prefix 검색만 가능함.
            - 레코드 락이나 넥스트 키락이 인덱스를 잠근 후에 테이블의 레코드를 잠그는 방식으로 구현된다.
    - B-Tree 인덱스 성능에 영향을 주는 요소들
        - 컬럼 크기
            - 페이지 크기의 디폴트는 16KB임. 인덱스 키가 16B, 자식노드 주소 크기가 12B라면 하나의 row가 28B이고 결국 한 페이지에는 16*1024B/28B = 585개의 키를 저장 가능하다.
            - 페이지당 키 값 갯수가 많을수록 적은 페이지를 읽으므로 빠르다.
        - Depth
            - 깊이가 깊을수록 랜덤IO 증가하므로 성능 감소.
        - Cardinality
            - 카디널리티가 클수록 중복된 키의 갯수는 적어진다.
            - 카디널리티가 작을수록 불필요한 데이터를 읽는 경우가 많아지므로 성능 감소한다.
            - 인덱스 키 구성 순서는 카디널리티 크기의 내림차순으로 구성하는게 합리적이다.
        - 레코드 수
            - 읽어야하는 레코드 수가 전체의 25%를 넘어선다면, 인덱스보다는 풀 스캔 후 필터링하는게 더 이득이다.
    - B-Tree를 이용한 읽기 방식
        - 인덱스 레인지 스캔
            - 검색해야 할 인덱스의 범위가 정해졌을 때 사용하는 방식
            - 루트-브랜치-리프 노드 순으로 인덱스 탐색 시작 지점 찾고, 리프 노드의 레코드 스캔(순서대로 탐색) → 리프 노드에서 얻은 레코드 주소로 랜덤IO를 하기 때문에 레코드가 많으면 성능이 오히려 떨어짐
            - 리프 노드를 스캔하다가 페이지 끝에 다다르면 리프 노드 간의 링크로 다음 페이지 이어서 스캔
            - 커버링 인덱스
                - 레코드까지 읽지 않고, 리프 노드까지의 인덱스 정보만으로 처리되는 쿼리 → 성능 향상
        - 인덱스 풀 스캔
            - 인덱스의 처음부터 끝까지 모두 읽는 방식
            - 쿼리의 조건절이 인덱스를 사용하지 못하는 경우에 사용된다.
            - 테이블 풀 스캔보다는 효율적이다.
        - Loose 인덱스 스캔
            - 인덱스 레인지 스캔과 비슷하지만, 리프 노드에서 필요하지 않은 인덱스 키 값은 스킵하고 이어서 탐색한다.
            - `GROUP BY`, `MAX`, `MIN` 등의 함수에 최적화할 때 사용한다.
        - 인덱스 스킵 스캔
            - 인덱스 컬럼 자체를 스킵한다.
            - 한계점
                - 커버링 인덱스여야함.
                - 스킵할 컬럼의 카디널리티가 커야함.
- PK를 따로 INT형으로 만드는 이유
    - Int형은 char 형보다 적은 바이트로 많은 유니크한 키값을 나타낼 수 있음 → 한 page 단위에 더 많은 index의 키 값들이 들어감 → 결과적으로 더 적은 page를 탐색하게된다.
    - 조회 시 연산에도 이점을 갖음 (like보다는 비교연산자가 빠르기 때문에)
    - char 형은 공백도 1바이트로 취급하기 때문에 메모리 효율 낮고, 무결성에 악영향을 준다.
- FK에 index가 걸릴까?
    - DB by DB 이다. MySQL은 FK를 걸면 FK에 대한 인덱스를 걸어준다.
- FK를 사용하지 않는 이유
    - FK의 목적은 데이터의 무결성과 정규화를 위해서 사용함.
    - 무결성 검사 등의 원인이 성능 저하를 야기함.
    - 무결성, 정합성 ↔ 개발 안정성, 확장성, 성능 간의 트레이드 오프
    - FK 대신에 인덱스를 사용하여 극복
- index가 항상 검색에만 사용되는 것은 아니다.
    
    [MySQL :: MySQL 8.0 Reference Manual :: 8.3.1 How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)
    
    > To find the rows matching a `WHERE` clause quickly.
    > 
    
    > To eliminate rows from consideration. If there is a choice between multiple indexes, MySQL normally uses the index that finds the smallest number of rows (the most [selective](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_selectivity) index).
    > 
    
    > If the table has a multiple-column index, any leftmost prefix of the index can be used by the optimizer to look up rows. For example, if you have a three-column index on `(col1, col2, col3)`, you have indexed search capabilities on `(col1)`, `(col1, col2)`, and `(col1, col2, col3)`. For more information, see [Section 8.3.6, “Multiple-Column Indexes”](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html).
    > 
    
    > To retrieve rows from other tables when performing joins. MySQL can use indexes on columns more efficiently if they are declared as the same type and size. In this context, `[VARCHAR](https://dev.mysql.com/doc/refman/8.0/en/char.html)` and `[CHAR](https://dev.mysql.com/doc/refman/8.0/en/char.html)` are considered the same if they are declared as the same size. For example, `VARCHAR(10)` and `CHAR(10)` are the same size, but `VARCHAR(10)` and `CHAR(15)` are not.
    > 
    > 
    > For comparisons between nonbinary string columns, both columns should use the same character set. For example, comparing a `utf8mb4` column with a `latin1` column precludes use of an index.
    > 
    > Comparison of dissimilar columns (comparing a string column to a temporal or numeric column, for example) may prevent use of indexes if values cannot be compared directly without conversion. For a given value such as `1` in the numeric column, it might compare equal to any number of values in the string column such as `'1'`, `' 1'`, `'00001'`, or `'01.e1'`. This rules out use of any indexes for the string column.
    > 
    
    > To find the `[MIN()](https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html#function_min)` or `[MAX()](https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html#function_max)` value for a specific indexed column ***`key_col`***. This is optimized by a preprocessor that checks whether you are using `WHERE ***key_part_N*** = ***constant***` on all key parts that occur before ***`key_col`*** in the index. In this case, MySQL does a single key lookup for each `[MIN()](https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html#function_min)` or `[MAX()](https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html#function_max)` expression and replaces it with a constant. If all expressions are replaced with constants, the query returns at once. For example:
    > 
    > 
    > `SELECT MIN(*key_part2*),MAX(*key_part2*)
    >   FROM *tbl_name* WHERE *key_part1*=10;`
    > 
    
    > To sort or group a table if the sorting or grouping is done on a leftmost prefix of a usable index (for example, `ORDER BY ***key_part1***, ***key_part2***`). If all key parts are followed by `DESC`, the key is read in reverse order. (Or, if the index is a descending index, the key is read in forward order.) See [Section 8.2.1.16, “ORDER BY Optimization”](https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html), [Section 8.2.1.17, “GROUP BY Optimization”](https://dev.mysql.com/doc/refman/8.0/en/group-by-optimization.html), and [Section 8.3.13, “Descending Indexes”](https://dev.mysql.com/doc/refman/8.0/en/descending-indexes.html).
    > 
    
    > In some cases, a query can be optimized to retrieve values without consulting the data rows. (An index that provides all the necessary results for a query is called a [covering index](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_covering_index).) If a query uses from a table only columns that are included in some index, the selected values can be retrieved from the index tree for greater speed:
    > 
    > 
    > `SELECT *key_part3* FROM *tbl_name*WHERE *key_part1*=1`
    > 
- index 키 크기에는 제한이 있다.
    - 3072Byte의 제한이 있다.
    - 속성의 일부에만 인덱스를 걸 수 있다.
    - 인덱스 속성의 크기는 작을수록 높은 성능을 낸다.
    
    [MySQL8.0 Error Code: 1071. Specified key was too long 해결법](https://leezzangmin.tistory.com/36)
    
- 같은 테이블에서 두 개의 다른 컬럼에 대한 인덱스를 사용하면 서로 영향을 줄까?
    
    [InnoDB의 레코드락에 대한 궁금점](https://leezzangmin.tistory.com/37)
    
    - select문을 for update로 실행하면 IX 락이 걸림 → row와 인덱스 모두에 락이 걸리기 때문에 대기 상태로 빠지는 듯.
- Index 관련 연산 디테일
    
    [MySQL :: MySQL 5.7 Reference Manual :: 14.13.1 Online DDL Operations](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html#online-ddl-index-operations)
    
    - `CREATE INDEX` 가 적용되는 테이블에 트랜잭션이 진행 중이라면, 모든 트랜잭션이 끝난 후에 인덱스가 생성된다.
    - 새로운 세컨더리 인덱스 생성 완료 시점에 uncommitted 데이터는 추가되지 않는다.

## ❓ 질문

## 🗂️ 참고자료

[MySQL Online-DDL](https://medium.com/daangn/mysql-online-ddl-faf47439084c)

[MySQL :: MySQL 8.0 Reference Manual :: 15.12.8 Online DDL Limitations](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-limitations.html)

[Foreign Key에도 index가 걸릴까?](https://ivvve.github.io/2020/07/08/server/rdb/is-fk-indexed/)

[MySQL :: MySQL 8.0 Reference Manual :: 8.3.1 How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)

[우리는 PK 에 왜 int 형 데이터를 고집할까](https://leo-bb.tistory.com/83)

[[mysql] 인덱스 정리 및 팁](https://jojoldu.tistory.com/243)

인덱스 요약 + 인덱스로 페이징 성능 개선 팁

[MySQL8.0 Error Code: 1071. Specified key was too long 해결법](https://leezzangmin.tistory.com/36)

[MySQL :: MySQL 5.7 Reference Manual :: 14.13.1 Online DDL Operations](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html#online-ddl-index-operations)
