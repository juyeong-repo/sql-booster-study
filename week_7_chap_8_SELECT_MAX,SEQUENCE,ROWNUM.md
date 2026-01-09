# 

- 채번: 키 값의 종류 중 하나인 ID나, 문서번호를 부여하는 것을 말함.

## SELECT ~ MAX

- 성능 이슈와, 잠재적인 오류 보유
- 동시에 여러 명이 사용할 경우를 고려해야 한다
- 여러 명이 동시에 N개의 세션에서 사용할 때 중복 오류가 발생할 가능성이 있다

### (1) REQ_DT를 이용한 SELECT~MAX

```sql
(Oracle)
SELECT /*+ GATEHR_PLAN_STATISTICS */
		 	 MAX(T1.PO_NO)
FROM   T_PO T1
WHERE  T1.REQ_DT >= TO_DATE('20170302', 'YYYYMMDD')
AND    T1.REQ_DT < TO_DATE('20170302', 'YYYYMMDD') + 1;
```

- 위 쿼리의 실행계획 확인시, **인덱스가 없는 경우** T_PO에 table full scan이 일어난다.
- 인덱스가 없는 상황에서 계속해서 데이터가 쌓이다 보면 심각한 성능 문제가 발생한다.

### (2) INDEX RANGE SCAN(MIN/MAX)

- 인덱스를 이용해 최댓값이나 최솟값을 빠르게 구하는 오퍼레이션이다.
    - 인덱스 리프 블록에서 조건에 해당하는 값 중에, 제일 앞이나, 제일 뒤 한 건만 읽어서 처리한다.
    - 리프 블록은 인덱스 키 값으로 정렬되어 있어서, 결과적으로 ‘MIN/MAX’ 집계 함수를 적용한 것과 같은 결과를 얻을 수 있다.
- (MIN/MAX)가 없는 INDEX RANGE SCAN은 조건에 해당하는 리프 데이터를 **모두** 읽는다.
    - 반면, 있는 경우 리프 데이터의 제일 앞 or 제일 뒤 **한 건만** 읽는다. → 성능 훌륭

## 채번 테이블

```sql
- 구매오더채번(T_PO_NUM)테이블 생성

CREATE TABLE T_PO_NUM (
	BAS_YMD VARCHAR(8) NOT NULL,
	LST_PO_NO VARCHAR2(40) NOT NULL
)

CREATE UNIQUE INDEX PK_T_PO_NUM ON T_PO_NUM (BAS_YMD);

ALTER TABLE T_PO_NUM
	ADD CONSTRAINT PK_T_PO_NUM PRIMARY KEY (BAS_YMD) USING INDEX;
	
	
- 이후 구매오더채번 테이블 이용한 채번 쿼리 작성
```

- 장점
    - 채번 테이블의 기준일자가 PK이므로 추가로 인덱스를 생성할 필요가 없다
    - 일자별로 한 건의 데이터만 존재하므로, 단 한 건의 데이터만 읽으면 채번할 수 있다
        - INDEX RANGE SCAN(MIN/MAX)를 고려할 필요가 없다
- 단점
    - 중복 오류 (최초에만 발생)
        - 해당 일자에 최초 채번이 동시에 실행되면 중복 오류가 발생할 수 있다
            
            ⇒ 해결 방법
            
            - 다음날 채번할 데이터를 미리 만들어 둔다.
            - 배치 작업을 등록해 처리하거나,
            - 오늘 첫 번 째 채번이 이루어질 때 내일의 0번째 채번 데이터를 같이 insert한다.
            - 또는 몇 년간의 채번 기초 데이터를 미리 만들어 둔다.
    - 동시성 저하
        - 동시 채번이 발생하면 후행 채번은 선행 세션의 트랜잭션이 완료할 때까지 대기한다
            
            ⇒ 해결 방법
            
            - 채번 과정을 독립된 트랜잭션으로 처리한다
            - 오라클의 경우 시퀀스(Sequence) 개체 사용을 권장한다
            - 오라클의 AUTONOMOUS_TRANSACTION 옵션의 사용자 정의 함수를 사용한다
    - 관리 비용
        - 채번 테이블을 추가로 생성해서 관리해야 하는 관리의 부담이 존재한다
            
            ⇒ 해결 방법
            
            - 채번이 필요한 모든 곳에 채번 테이블 생성보다는, 어느정도 통합된 구조의 채번 테이블을 만든다
    

# 8.4 시퀀스와 ROWNUM

- 오라클에 시퀀스(Sequence) 객체가 있음 (MySQL ⇒ AUTO_INCREMENT)
    - 시퀀스 사용시 차례대로 증가하는 숫자 값을 얻어낼 때 아주 편리함
    - 특정 테이블의 PK 값으로 자주 사용됨
    - 무조건적인 사용보다는, 명호가하고 간단한 PK 후보가 있다면 해당 컬럼을 PK로 잡아주는 게 좋음
    - 시퀀스는 트랜잭션의 ‘COMMIT, ROLLBACK’과는 무관하게 처리됨
    - 주의사항
        - INSERT된 시퀀스 값을 이용해 후행 처리시 SELECT~MAX를 절대 사용해선 안된다
            - ex. 주문 데이터 생성 후 바로 주문 상세 생성을 위해 방금 입력한 주문 데이터의 PK값을 select~max로 가져오는 케이스 (X)
            - 테이블을 불필요하게 한 번 더 접근함, 잘못된 시퀀스 값을 가져올 수 있음. (트랜잭션 이슈)
            - 오라클 - NEXTVAL보다는 CURRVAL 사용
        - MySQL/MSSQL의 경우
            - INSERT 시점에 시퀀스 형태 컬럼에 자동으로 증가한 값이 부여된다
            - MySQL - LAST_INSERT_ID() / MSSQL - @@IDENTITY

- ROWNUM (MySQL ⇒ limit)
    - 최근 N건의 데이터만 조회할 때 사용
    - 이 때도 인덱스가 걸려있으면 성능이 훨씬 좋아진다.
