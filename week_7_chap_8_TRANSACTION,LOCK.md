📌오라클 → PostgreSql 15로 변환

- 8.1 트랜잭션
    
    ### 8.1.1 트랜잭션(Transaction)이란?
    
    반드시 한 번에 처리되어야 하는 논리적인 작업 단위
    
    절대 쪼개서 실행할 수 없는 작업의 최소 단위
    
    (예, 계좌 이체)
    
    ### 트랜잭션을 종료하는 명령어
    
    COMMIT: 트랜잭션 과정 중에 변경된 데이터를 모두 반영하고 종료하는 명령어
    
    ROLLBACK: 트랜잭션 과정에서 진행된 작업을 모두 취소하고 종료하는 명령어
    
    ### 8.1.2  트랜잭션 테스트
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.2 SQL1
    -- ************************************************
    
    	-- 계좌 테이블 및 계좌 데이터 생성
    	-- 계좌 테이블을 생성
    	create table M_ACC
    	(
    		ACC_NO VARCHAR(40) not null,
    		ACC_NM VARCHAR(100),
    		BAL_AMT numeric(18,3)
    	);
    	
    	-- 기본 키 제약 조건 추가
    	ALTER TABLE M_ACC
    	    ADD CONSTRAINT PK_M_ACC PRIMARY KEY (ACC_NO);
    	
    	-- 테스트 데이터를 생성
    	insert into M_ACC (ACC_NO, ACC_NM, BAL_AMT)
    	values 
    		('ACC1', '1번계좌', 3000),
    		('ACC2', '2번계좌', 500),
    		('ACC3', '3번계좌', 0);
    ```
    
    ### (1) 정상적인 계좌이체
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.2 SQL2
    -- ************************************************
    
    	-- 계좌이체 – ACC1에서 ACC2로 500원 이체
    	update M_ACC
    	set BAL_AMT = BAL_AMT - 500
    	where ACC_NO = 'ACC1';
    	
    	update M_ACC
    	set BAL_AMT = BAL_AMT + 500
    	where ACC_NO = 'ACC2';
    	
    	commit;
    ```
    
    ### (2) 비정상적인 계좌이체 - 수신 계좌가 없는 경우
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.2 SQL3
    -- ************************************************
    
    	-- 계좌이체 – ACC1에서 ACC4로 500원 이체
    	-- ACC1에서 500원 출금
    	update M_ACC
    	set BAL_AMT = BAL_AMT - 500
    	where ACC_NO = 'ACC1';
    	
    	-- ACC4로 500원 입금 (ACC4가 존재하지 않으면 영향 없음)
    	update M_ACC
    	set BAL_AMT = BAL_AMT + 500
    	where ACC_NO = 'ACC54';
    	
    	-- 계좌 상태 조회
    	select * from M_ACC;
    
    -- ************************************************
    -- PART III - 8.1.2 SQL4
    -- ************************************************
    
    	-- 계좌이체 – ACC1에서 ACC4로 500원 이체, ROLLBACK 처리
    	rollback;
    	
    	-- 롤백 후 계좌 상태 재확인
    	select * from M_ACC;
    
    -- ************************************************
    -- PART III - 8.1.2 SQL5
    -- ************************************************
    
    	-- 계좌이체 – 계좌존재여부 검증
    	select COALESCE(MAX('Y'), 'N')
    	where exists (
    		select 1 from M_ACC A where A.ACC_NO = 'ACC4'
    	);
    	
    	select case
    			 when exists (select 1 from M_ACC where ACC_NO = 'ACC4') then 'Y'
    			 else 'N'
    		   end;
    ```
    
    ### (3) 비정상적인 계좌이체 - 이체 금액이 잔액보다 큰 경우
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.2 SQL6
    -- ************************************************
    
    	-- 계좌이체 – ACC1에서 ACC3로 5000원 이체
    	-- ACC1에서 ACC3으로 5000원 이체
    	update m_acc
    	set bal_amt = bal_amt - 5000
    	where acc_no = 'ACC1';
    	
    	update m_acc
    	set bal_amt = bal_amt + 5000
    	where acc_no = 'ACC3';
    	
    	select * from m_acc;
    
    -- ************************************************
    -- PART III - 8.1.2 SQL7
    -- ************************************************
    	
    	-- 롤백 (트랜잭션 취소)
    	rollback;
    	
    	select * from m_acc;
    ```
    
    ### (4) 트랜잭션 중에 에러가 발생한 경우
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.2 SQL8
    -- ************************************************
    
    	-- 새로운 계좌 ACC4 추가
    	insert into m_acc (acc_no, acc_nm, bal_amt)
    	values ('ACC4', '4번계좌', 0);
    	
    	-- 이미 존재하는 ACC1을 다시 추가하려고 하면 오류 발생
    	insert into m_acc (acc_no, acc_nm, bal_amt)
    	values ('ACC1', '1번계좌', 0);
    
    	-- 현재 상태 확인 (에러 발생 전이면 ACC4만 들어감)
    	select * from m_acc;
    	
    	-- 트랜잭션 롤백: 위 INSERT 모두 취소
    	rollback;
    ```
    
    PostgreSQL은 기본적으로 **트랜잭션 내에서 하나라도 에러가 발생하면 전체 트랜잭션이 실패(ABORT)** 상태가 됩니다.
    
    - 트랜잭션은 한 번에 이루어져야 하는 작업 단위다. (더는 쪼갤 수 없는 작업 단위)
    - 트랜잭션은 COMMIT이나 ROLLBACK으로 종료가 이루어진다.
    - COMMIT으로 종료될 경우, 트랜잭션에서 변경된 데이터들은 모두 DB에 실제 반영된다.
    - ROLLBACK으로 종료될 경우, 트랜잭션 시작 이전으로 데이터들은 복구가 된다.
    - 트랜잭션 내에 에러가 발생했다고 해서 자동으로 ROLLBACK이 수행되지 않는다.
    
    ### 8.1.3 트랜잭션 고립화 수준 - READ COMMITTED
    
    하나의 트랜잭션에서 작업 중인 데이터가 다른 트랜잭션에 영향을 받지 않는 정도
    
    하나의 트랜잭션에서 작업 중인 데이터를 다른 트랜잭션이 어느 정도까지 접근할 수 있는지에 대한 정도
    
    고립화 수준을 가장 낮은 단계로 설정하면, 한 트랜잭션에서 변경 중인 데이터를 다른 트랜잭션에서 접근할 수 있다.
    
    고립화 수준을 높은 단계로 설정하면, 조회만 이루어진 데이터도다른 트랜잭션이 변경할 수 없게 한다.
    
    동시성: 데이터베이스를 동시에 여러 명이 사용할 수 있는 능력
    
    일반적으로 고립화 수준을 낮게 설정하면 동시성은 좋아진다.
    
    반대로 고립화 수준을 높게 설정할수록 동시성은 나빠진다.
    
    트랜잭션에서 불필요한 작업은 제거해 간결하게 만들고, SQL을 최적화해서 잘 구현한다면, 데이터의 정확성과 동시성 모두 확보할 수 있다.
    
    - READ UNCOMMITTED: 다른 트랜잭션이 아직 커밋하지 않은 데이터를 읽을 수 있음
    - READ COMMITTED: 커밋된 데이터만 읽을 수 있음
    - REPEATABLE READ: 트랜잭션 내에서 같은 쿼리는 항상 같은 결과를 반환
    - SERIALIZABLE READ: 트랜잭션이 직렬화된 순서대로 실행된 것처럼 동작
    
    ### (1) READ COMMITTED: UPDATE-SELECT 테스트
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL1
    -- ************************************************
    
    	-- UPDATE-SELECT 테스트 – 첫 번째 세션 SQL
    	select * from m_acc where acc_no = 'ACC1'; --ACC1의 잔액은 2500원
    	
    	update m_acc
    	set bal_amt = 5000
    	where acc_no = 'ACC1';
    	
    	select * from m_acc where acc_no = 'ACC1'; --ACC1의 잔액 5000원
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL2
    -- ************************************************
    
    	-- UPDATE-SELECT 테스트 – 두 번째 세션 SQL
    	select * from m_acc where acc_no = 'ACC1'; --ACC1의 잔액 2500원
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL3
    -- ************************************************
    
    	-- UPDATE-SELECT 테스트 – 첫 번째 세션 COMMIT 처리
    	COMMIT;
    ```
    
    ```sql
    	-- UPDATE-SELECT 테스트 – 두 번째 세션 SQL
    	select * from m_acc where acc_no = 'ACC1'; --ACC1의 잔액 5000원
    ```
    
    ### (2) READ COMMITTED: UPDATE-UPDATE 테스트
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL4
    -- ************************************************
    
    	-- UPDATE – UPDATE 테스트 – 첫 번째 세션 SQL
    	--현재 ACC1의 잔액은 5,000
    	update m_acc
    	set bal_amt = bal_amt - 500
    	where acc_no = 'ACC1';
    	
    	select * from m_acc where acc_no = 'ACC1';
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL5
    -- ************************************************
    
    	--UPDATE – UPDATE 테스트 – 두 번째 세션 SQL
    	-- 아직 첫 번째 세션의 UPDATE 문이 COMMIT 되지 않았으므로
    	-- 두 번째 세션에서는 첫 번째 세션의 UPDATE 이전 데이터가 조회된다.
    	select * from m_acc where acc_no = 'ACC1'; --ACC1의 잔액은 5,000원
    
    	-- 아래 SQL은 첫 번째 세션에 막혀 진행되지 못한다.
    	update m_acc
    	set bal_amt = bal_amt - 500
    	where acc_no = 'ACC1';
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL6
    -- ************************************************
    
    	-- UPDATE – UPDATE 테스트 – 첫 번째 세션 COMMIT
    	COMMIT;
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL7
    -- ************************************************
    
    	-- UPDATE – UPDATE 테스트 – 두 번째 세션 확인
    	COMMIT;
    	
    	select * from m_acc where acc_no = 'ACC1'; --ACC1의 잔액은 4,000원
    ```
    
    대기(WAIT) 상태: 선행 트랜잭션이 데이터를 변경하고 있으므로, 후행 트랜잭션이 데이터에 접근하지 못하고 기다리는 상태
    
    ### (3) READ COMMITTED: INSERT-INSERT 테스트 #1
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL8
    -- ************************************************
    
    	-- INSERT – INSERT 테스트 – 첫 번째 세션 ACC4 생성
    	insert into m_acc (acc_no, acc_nm, bal_amt)
    	values ('ACC4', '4번계좌', 0);
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL9
    -- ************************************************
    
    	-- INSERT – INSERT 테스트 – 두 번째 세션 ACC4 생성
    	insert into m_acc (acc_no, acc_nm, bal_amt)
    	values ('ACC4', '4번계좌', 0);
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL10
    -- ************************************************
    
    	-- INSERT – INSERT 테스트 – 첫 번째 세션 COMMIT
    	COMMIT;
    	
    	-- 두 번째 세션에서는 에러가 발생
    ```
    
    ### (4) READ COMMITTED: INSERT-INSERT 테스트 #2
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL11
    -- ************************************************
    
    	-- INSERT – INSERT 테스트 – 첫 번째 세션 ACC5 생성
    	insert into m_acc (acc_no, acc_nm, bal_amt)
    	values ('ACC5', '5번계좌', 0);
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL12
    -- ************************************************
    
    	-- INSERT – INSERT 테스트 – 두 번째 세션 ACC99 생성
    	insert into m_acc (acc_no, acc_nm, bal_amt)
    	values ('ACC99', '99번계좌', 0);
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3 SQL13
    -- ************************************************
    
    	-- INSERT – INSERT 테스트 – 두 번째 세션 ACC5 생성
    	insert into m_acc (acc_no, acc_nm, bal_amt)
    	values ('ACC5', '5번계좌', 0); -- 대기상태
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.1.3	 SQL14
    -- ************************************************
    	-- INSERT – INSERT 테스트 – 첫 번째 세션 COMMIT
    	COMMIT;
    	
    	-- 두 번째 세션에서는 중복 에러 발생
    ```
    
- 8.2 락(LOCK)
    
    ### 8.2.1 락(LOCK)
    
    데이터에 잠금을 걸어 놓는 장치
    
    대기 상태
    
    인라인ㅡ뷰와 비슷하다.
    
    SQL의 가장 윗부분에서 사용한다.
    
    WITH 절에서 정의된 SQL 블록들은 같은 SQL 내에서 테이블처럼 사용할 수 있다.
    
    ```sql
    -- ************************************************
    -- PART I - 4.3.1 SQL1
    -- ************************************************
    
    -- 고객, 아이템유형별 주문금액 구하기 – 인라인-뷰 이용
    select	T0.CUS_ID,
    		T1.CUS_NM,
    		T0.ITM_TP,
    		(select	A.BAS_CD_NM
    		 from	C_BAS_CD A
    		 where 	A.LNG_CD = 'KO'
    		 and	A.BAS_CD_DV = 'ITM_TP'
    		 and	A.BAS_CD = T0.ITM_TP) as ITM_TP_NM,
    		T0.ORD_AMT
    from	(
    			select	A.CUS_ID,
    					C.ITM_TP,
    					SUM(B.ORD_QTY * B.UNT_PRC) as ORD_AMT
    			from	T_ORD A
    			join	T_ORD_DET B on A.ORD_SEQ = B.ord_seq
    			join	M_ITM C on B.ITM_ID = C.itm_id
    			where	A.ORD_DT >= '2017-02-01'::DATE
    			and 	A.ORD_DT < '2017-03-01'::DATE
    			group by A.CUS_ID, C.ITM_TP
    		) T0
    join	M_CUS T1 on T1.CUS_ID = T0.cus_id
    order by T0.CUS_ID, T0.ITM_TP;
    
    -- ************************************************
    -- PART I - 4.3.1 SQL2
    -- ************************************************
    
    -- 고객, 아이템유형별 주문금액 구하기 – WITH~AS 이용
    -- 반복되는 인라인ㅡ뷰를 제거해 성능을 개선하거나, 가독성을 좋게 할 수 있음 (무조건 X)
    with T_CUS_ITM_AMT as (
    	select	A.CUS_ID,
    			C.ITM_TP,
    			SUM(B.ORD_QTY * B.UNT_PRC) as ORD_AMT
    	from	T_ORD A
    	join	T_ORD_DET B on A.ORD_SEQ = B.ORD_SEQ
    	join 	M_ITM C on B.ITM_ID = C.itm_id
    	where	A.ORD_DT >= '2017-02-01'::DATE
    	and 	A.ORD_DT < '2017-03-01'::DATE
    	group by A.CUS_ID, C.ITM_TP
    )
    select	T0.CUS_ID,
    		T1.CUS_NM,
    		T0.ITM_TP,
    		(select	A.BAS_CD_NM
    		 from	C_BAS_CD A
    		 where	A.LNG_CD = 'KO'
    		 and	A.BAS_CD_DV = 'ITM_TP'
    		 and	A.BAS_CD = T0.ITM_TP) as ITM_TP_NM,
    		T0.ORD_AMT
    from	T_CUS_ITM_AMT T0
    join	M_CUS T1 on T1.CUS_ID = T0.CUS_ID
    order by T0.CUS_ID, T0.ITM_TP;
    
    -- ************************************************
    -- PART I - 4.3.1 SQL3
    -- ************************************************
    
    -- 고객, 아이템유형별 주문금액 구하기, 전체주문 대비 주문금액비율 추가 – WITH~AS 이용
    -- 같은 테이블을 WITH 절마다 반복 사용하는 것은 주의
    with T_CUS_ITM_AMT as (
    	select	A.CUS_ID,
    			C.ITM_TP,
    			SUM(B.ORD_QTY * B.UNT_PRC) as ORD_AMT
    	from	T_ORD A
    	join	T_ORD_DET B on A.ORD_SEQ = B.ord_seq
    	join	M_ITM C on B.ITM_ID = C.ITM_ID
    	where	A.ORD_DT >= '2017-02-01'::DATE
    	and 	A.ORD_DT < '2017-03-01'::DATE
    	group by A.CUS_ID, C.ITM_TP
    ),
    T_TTL_AMT as ( -- T_CUS_ITM_AMT 사용 가능
    	select	SUM(ORD_AMT) as ORD_AMT
    	from 	T_CUS_ITM_AMT
    )
    select	T0.CUS_ID,
    		T1.CUS_NM,
    		T0.ITM_TP,
    		(select	A.BAS_CD_NM
    		 from	C_BAS_CD A
    		 where	A.LNG_CD = 'KO'
    		 and	A.BAS_CD_DV = 'ITM_TP'
    		 and	A.BAS_CD = T0.ITM_TP) as ITM_TP_NM,
    		T0.ORD_AMT,
    		(ROUND(T0.ORD_AMT / T2.ORD_AMT * 100, 2))::text || '%' as ORD_AMT_RT
    from	T_CUS_ITM_AMT T0
    join	M_CUS T1 on T1.CUS_ID = T0.CUS_ID
    join	T_TTL_AMT T2 on true
    order by ROUND(T0.ORD_AMT / T2.ORD_AMT * 100, 2) desc;
    ```
    
    ### 4.3.2 WITH 절을 사용한 INSERT
    
    ```sql
    -- ************************************************
    -- PART I - 4.3.2 SQL1
    -- ************************************************
    
    -- 주문금액 비율 컬럼 추가
    alter table S_CUS_YM add ORD_AMT_RT NUMERIC(18,3);
    
    -- ************************************************
    -- PART I - 4.3.2 SQL2
    -- ************************************************
    
    -- WITH~AS 절을 사용한 INSERT문
    insert into S_CUS_YM (BAS_YM, CUS_ID, ITM_TP, ORD_QTY, ORD_AMT, ORD_AMT_RT)
    with T_CUS_ITM_AMT as (
    	select	TO_CHAR(A.ORD_DT, 'YYYYMM') as BAS_YM,
    			A.CUS_ID,
    			C.ITM_TP,
    			SUM(B.ORD_QTY) as ORD_QTY,
    			SUM(B.ORD_QTY * B.UNT_PRC) as ORD_AMT
    	from	T_ORD A
    			join T_ORD_DET B on A.ORD_SEQ = B.ORD_SEQ
    			join M_ITM C on B.ITM_ID = C.ITM_ID
    	where	A.ORD_DT >= '2017-04-01'::DATE
    	and		A.ORD_DT < '2017-05-01'::DATE
    	group by TO_CHAR(A.ORD_DT, 'YYYYMM'), A.CUS_ID, C.ITM_TP
    )
    , T_TTL_AMT as (
    	select	SUM(A.ORD_AMT) as ORD_AMT
    	from	T_CUS_ITM_AMT A
    )
    select	T0.BAS_YM,
    		T0.CUS_ID,
    		T0.ITM_TP,
    		T0.ORD_QTY,
    		T0.ORD_AMT,
    		ROUND(T0.ORD_AMT / T2.ORD_AMT * 100, 2) as ORD_AMT_RT
    from	T_CUS_ITM_AMT T0
    		cross join T_TTL_AMT T2
    		join M_CUS T1 on T1.CUS_ID = T0.CUS_ID;
    ```
    
    ### 8.2.2 SELECT~FOUR UPDATE
    
    ‘READ COMMITTED’는 변경된 데이터에 로우 단위로 변경 락을 생성한다.
    
    ‘READ COMMITTED’ 수준에서 데이터 일관성을 확보하려면 ‘SELECT~FOR UPDATE’를 활용해야 한다.
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.2 SQL1
    -- ************************************************
    
    	-- 동시에 두 개의 세션이 ACC1의 잔액에서 4,000원씩을 출금하려고 한다.
    	--첫 번째 세션
    	--1.ACC1에서 4,000원을 출금하려고 한다.
    	select bal_amt from m_acc where acc_no = 'ACC1'; -- 4,000 조회
    ```
    
    ```sql
    			--두 번째 세션
    			--2.ACC1에서 4,000원을 출금하려고 한다.
    			select bal_amt from m_acc where acc_no = 'ACC1'; -- 4,000 조회
    ```
    
    ```sql
    	--첫 번째 세션
    	--IF BAL_AMT >= 4,000 THEN UPDATE
    	--3.잔액이 4,000원이 있으므로 4,000원을 출금처리.
    	update m_acc
    	set bal_amt = bal_amt - 4000
    	where acc_no = 'ACC1';
    	
    	
    	--첫 번째 세션
    	--4.잔액이 0원이 되어 있다.
    	select bal_amt from m_acc where acc_no = 'ACC1';
    ```
    
    ```sql
    			--두 번째 세션
    			--IF BAL_AMT >= 4,000 THEN UPDATE
    			--5.잔액이 4,000원이 있으므로 4,000원을 출금처리.
    			--첫 번째 세션에 의해 대기 상태에 빠진다.
    			update m_acc
    			set bal_amt = bal_amt - 4000
    			where acc_no = 'ACC1';
    ```
    
    ```sql
    	--첫 번째 세션
    	--6.COMMIT처리.
    	COMMIT;
    ```
    
    ```sql
    			--두 번째 세션
    			--7. 잔액이 마이너스 4,000원이 되어 있다.
    			select bal_amt from m_acc where acc_no = 'ACC1';
    			
    			--두 번째 세션
    			--8. COMMIT처리
    			COMMIT;
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.2 SQL2
    -- ************************************************
    
    	-- ACC1의 잔액을 4,000원으로 변경
    	update m_acc
    	set bal_amt = 4000
    	where acc_no = 'ACC1';
    	
    	commit;
    ```
    
    ### ‘SELECT~FOR UPDATE’ 이용
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.2 SQL3
    -- ************************************************
    
    	--동시에 두 개의 세션이 ACC1의 계좌에서 4,000원씩을 출금하려고 한다. – SELECT ~ FOR UPDATE사용.
    
    	-- 첫 번째 세션
    	--1. ACC1에서 4,000원을 출금하려고 한다.
    	select bal_amt from m_acc
    	where acc_no = 'ACC1'
    	for update;
    ```
    
    ```sql
    			--두 번째 세션
    			--2. ACC1에서 4,000원을 출금하려고 한다.
    			select bal_amt from m_acc
    			where acc_no = 'ACC1'
    			for update;
    			-- 대기 상태에 빠졌다가 첫 번째 세션이 COMMIT된 후
    			-- 0원이 조회된다.
    ```
    
    ```sql
    	-- 첫 번째 세션
    	--3. 잔액이 4,000원이 있으므로 4,000원을 출금처리.
    	-- IF BAL_AMT >= 4,000 THEN UPDATE
    	update m_acc
    	set bal_amt = bal_amt - 4000
    	where acc_no = 'ACC1';
    
    	
    	-- 첫 번째 세션
    	--4. 잔액이 0원이 되어 있다.
    	select bal_amt from m_acc
    	where acc_no = 'ACC1';
    
    	-- 첫 번째 세션
    	--5. COMMIT처리.
    	COMMIT;
    
    			--두 번째 세션
    			-- IF BAL_AMT >= 4,000 THEN UPDATE
    			-- 잔액이 4,000보다 작으므로 출금 불가.
    
    			--두 번째 세션
    			ROLLBACK;
    ```
    
    ‘SELECT~FOR UPDATE’는 데이터 조회 시점부터 변경 락을 생성해 데이터의 일관성을 확보할 수 있게 해준다.
    
    ‘NOWAIT’와 ‘WAIT SECONDS’ 옵션
    
    ### 8.2.3 대기(WAIT) 상태
    
    대기 상태의 세션이 많아지면 시스템이 장애로 빠질 수 있다.
    
    폴링은 WAS에서 특정 수만큼의 세션을 데이터베이스와 미리 연결해 놓고 여러 명의 사용자가 세션을 공유해 사용하는 방법이다.
    
    시스템의 동시성을 높이려면 불필요한 ‘SELECT~FOR UPDATE’ 사용을 피하고, 트랜잭션은 항상 제대로 종료해야 한다.
    
    ### 8.2.4 데드락(DEAD-LOCK, 교착상태)
    
    첫 번째 세션이 두 번째 세션의 작업이 끝나기를 기다리고 있고 두 번쨰 세션도 첫 번째 세션의 작업이 끝나기를 기다리는 상태
    
    대기 상태는 데이터의 정확성을 위해 기다리는 (정상적인) 상태지만, 데드락은 더는 트랜잭션을 진행할 수 없는 상태
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.4 SQL1
    -- ************************************************
    
    	-- ACC1, ACC2의 잔액 초기화
    	update m_acc
    	set bal_amt = 5000
    	where acc_no in ('ACC1', 'ACC2');
    	
    	commit;
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.4 SQL2
    -- ************************************************
    
    	-- 데드락 테스트 – 두 개의 세션에서 계좌이체 실행
    	--첫 번째 세션, ACC1->ACC2 2,000원 이체
    	--1.ACC1의 잔액 확인
    	select bal_amt from m_acc where acc_no = 'ACC1' for update;
    
    	--첫 번째 세션
    	--2.ACC1에서 잔액 마이너스
    	--(잔액이 이체금액 이상이면 이체 수행)
    	update m_acc set bal_amt = bal_amt - 2000 where acc_no = 'ACC1';
    ```
    
    ```sql
    			--두 번째 세션, ACC2->ACC1 3,000원 이체
    			--두 번째 세션
    			--3.ACC2의 잔액 확인
    			select bal_amt from m_acc where acc_no = 'ACC2' for update;
    
    			--두 번째 세션
    			--4. ACC2에서 잔액 마이너스
    			--IF BAL_AMT >= 3,000 THEN UPDATE
    			update m_acc set bal_amt = bal_amt - 3000 where acc_no = 'ACC2';
    ```
    
    ```sql
    	--첫 번째 세션
    	--5. ACC2의 잔액 플러스
    	--두 번째 세션 3번,4번 SQL에 의해 대기에 빠진다.
    	update m_acc set bal_amt = bal_amt + 2000 where acc_no = 'ACC2';
    ```
    
    ```sql
    			--두 번째 세션
    			--6.ACC1의 잔액 플러스
    			--첫 번째 세션 1번,5번 SQL에 의해 대기에 빠진다.
    			update m_acc set bal_amt = bal_amt + 3000 where acc_no = 'ACC1';
    ```
    
    ### 동시에 락을 생성
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.5 SQL1
    -- ************************************************
    
    	-- ACC1->ACC2 2,000원 계좌이체 트랜잭션 SQL
    	-- 1.ACC1, ACC2의 잔액 확인
    	select acc_no, bal_amt
    	from m_acc
    	where acc_no in ('ACC1', 'ACC2')
    	for update;
    
    	-- 첫 번째 세션
    	-- 2.ACC1에서 잔액 마이너스
    	-- (잔액이 이체금액 이상이면 이체 수행)
    	update m_acc
    	set bal_amt = bal_amt - 2000
    	where acc_no = 'ACC1';
    ```
    
    ```sql
    			-- 두 번째 세션, ACC2->ACC1 3,000원 이체
    
    			-- 두 번째 세션
    			-- 3.ACC1, ACC2의 잔액 확인
    			-- 첫 번째 세션 1번 SQL로 인해 대기 상태에 빠진다.
    			select acc_no, bal_amt
    			from m_acc
    			where acc_no in ('ACC1', 'ACC2')
    			for update;
    
    ```
    
    ```sql
    	-- 첫 번째 세션
    	-- 4. ACC2의 잔액 플러스
    	update m_acc
    	set bal_amt = bal_amt + 2000
    	where acc_no = 'ACC2';
    
    	-- 첫 번째 세션
    	COMMIT;
    ```
    
    ```sql
    			-- 두 번째 세션
    			-- 5. ACC2에서 잔액 마이너스
    			-- IF BAL_AMT >= 3,000 THEN UPDATE
    			update m_acc
    			set bal_amt = bal_amt - 3000
    			where acc_no = 'ACC2';
    
    			-- 두 번째 세션
    			-- 6.ACC1의 잔액 플러스
    			update m_acc
    			set bal_amt = bal_amt + 3000
    			where acc_no = 'ACC1';
    
    			-- 두 번째 세션
    			COMMIT;
    ```
    
    ### 8.2.5 트랜잭션 최소화
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.5 SQL1
    -- ************************************************
    
    	-- ACC1->ACC2 2,000원 계좌이체 트랜잭션 SQL
    	-- 1.ACC1, ACC2의 잔액 확인
    	select acc_no, bal_amt
    	from m_acc
    	where acc_no in ('ACC1', 'ACC2')
    	for update;
    	
    	-- 2.ACC1의 잔액이 이체금액 이상이면 이체 수행
    	update m_acc
    	set bal_amt = bal_amt - 2000
    	where acc_no = 'ACC1';
    	
    	-- 3. ACC2의 잔액 플러스
    	update m_acc
    	set bal_amt = bal_amt + 2000
    	where acc_no = 'ACC2';
    	commit;
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.5 SQL2
    -- ************************************************
    
    	-- ACC1->ACC2 2,000원 계좌이체 트랜잭션 SQL
    	-- 1.ACC1, ACC2의 잔액 확인
    	select T1.ACC_NO, T1.BAL_AMT
    	from M_ACC T1
    	where T1.ACC_NO in ('ACC1', 'ACC2')
    	for update;
    
    	-- 2.ACC1의 BAL_AMT가 이체 금액보다 작으면 ROLLBACK처리(잔액이 부족합니다.)
    
    	-- 3.ACC2가 존재하지 않는다면 ROLLBACK처리(수신 계좌가 존재하지 않습니다.)
    
    	-- 4.ACC1과 ACC2의 잔액을 동시에 처리
    	update M_ACC T1
    	set BAL_AMT = BAL_AMT +
    		case
    			when T1.ACC_NO = 'ACC1' then -1 + 2000
    			when T1.ACC_NO = 'ACC2' then 1+ 2000
    			else 0
    		end
    	where T1.ACC_NO in ('ACC1', 'ACC2');
    	commit;
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.5 SQL3
    -- ************************************************
    
    	-- ACC1->ACC2 2,000원 계좌이체 트랜잭션 SQL – 한 문장으로 처리
    	update M_ACC
    	set BAL_AMT = BAL_AMT +
    		case
    			when ACC_NO = 'ACC1' then -2000
    			when ACC_NO = 'ACC2' then 2000
    		end
    	where ACC_NO in ('ACC1', 'ACC2')
    	  and BAL_AMT >=
    	  	case
    	  		when ACC_NO = 'ACC1' then 2000
    	  		when ACC_NO = 'ACC2' then 0
    	  	end;
    
    	-- UPDATE된 건수가 두 건이면 COMMIT.
    	-- UPDATE된 건수가 두 건이 아니면 ROLLBACK
    	COMMIT;
    ```
    
    ### 8.2.6 방어 로직
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.6 SQL1
    -- ************************************************
    	update M_ACC set BAL_AMT = 3000;
    	commit;
    ```
    
    ```sql
    -- ************************************************
    -- PART III - 8.2.6 SQL1
    -- ************************************************
    	update M_ACC set BAL_AMT = 3000;
    	commit;
    
    -- ************************************************
    -- PART III - 8.2.6 SQL2
    -- ************************************************
    	-- 1.ACC1, ACC2의 잔액 확인(ACC1과 ACC2 모두 3000원이 조회된다.)
    	-- ACC1의 잔액은 @FROM_BAL_AMT에, ACC2의 잔약은 @TO_BAL_AMT에 저장한다.
    	select T1.ACC_NO, T1.BAL_AMT
    	from M_ACC T1
    	where T1.ACC_NO in ('ACC1', 'ACC2'); -- SELECT~FOR UPDATE를 사용하지 않는다.
    
    	-- 2.ACC1의 BAL_AMT가 이체 금액보다 작으면 ROLLBACK처리(잔액이 부족합니다.)
    
    	-- 3.ACC2가 존재하지 않는다면 ROLLBACK처리(수신 계좌가 존재하지 않습니다.)
    
    	-- 4.ACC1과 ACC2의 잔액을 동시에 처리
    	update M_ACC
    	set BAL_AMT = BAL_AMT +
    		case when ACC_NO = 'ACC1' then -1 * 2000
    			 when ACC_NO = 'ACC2' then 1 * 2000 end
    	where ACC_NO in ('ACC1', 'ACC2')
    	and BAL_AMT = case when ACC_NO = 'ACC1' then 3000 --@FROM_BAL_AMT 값을 사용
    					   when ACC_NO = 'ACC2' then 3000 --@TO_BAL_AMT  값을 사용
    				  end;
    
    	-- 5. UPDATE된 건수가 2건 일때만 COMMIT처리.
    	commit;
    ```
    
    ### 8.2.7 불필요한 트랜잭션의 분리
    
    트랜잭션과 주요 연관성이 떨어지는 작업을 분리하면 트랜잭션의 실행 시간을 줄일 수 있다.
