📌오라클 → PostgreSql 15로 변환

- 4.1 서브쿼리
    
    ### 4.1.1 서브쿼리의 종류
    
    서브쿼리는 성능이 좋지 못할 수 있다.
    
    가장 조심할 부분은 무분별한 서브쿼리 남발이다.
    
    - SELECT 절의 단독 서브쿼리
    - SELECT 절의 상관 서브쿼리
    - WHERE 절의 단독 서브쿼리
    - WHERE 절의 상관 서브쿼리
    - 인라인ㅡ뷰
    
    단독 서브쿼리 : 메인 SQL과 상관없이 실행 할 수 있는 서브쿼리
    
    상관 서브쿼리 : 메인 SQL에서 값을 받아서 처리해야 하는 서브쿼리
    
    (메인 SQL : 서브쿼리를 제외한 나머지 SQL)
    
    ### 4.1.2 SELECT 절의 단독 서브쿼리
    
    ```sql
    -- ************************************************
    -- PART I - 4.1.2 SQL1
    -- ************************************************
    
    -- 17년8월 총 주문금액 구하기 – SELECT절 단독 서브쿼리
    select	TO_CHAR(T1.ORD_DT, 'YYYYMMDD') as ORD_YMD,
    		SUM(T1.ORD_AMT) as ORD_AMT,
    		(
    			select	SUM(A.ORD_AMT)
    			from	T_ORD A
    			where 	A.ORD_DT >= DATE '2017-08-01'
    			and		A.ORD_DT < DATE '2017-09-01'
    		) as TOTAL_ORD_AMT -- SELECT 절의 단독 서브쿼리
    from	T_ORD T1
    where	T1.ORD_DT >= DATE '2017-08-01'
    and		T1.ORD_DT < DATE '2017-09-01'
    group by TO_CHAR(T1.ORD_DT, 'YYYYMMDD');
    
    -- ************************************************
    -- PART I - 4.1.2 SQL2
    -- ************************************************
    
    -- 17년8월 총 주문금액, 주문일자의 주문금액비율 구하기 – SELECT절 단독 서브쿼리
    -- 주문금액 비율 = 주문일자별 주문금액(ORD_AMT) / 17년8월 주문 총 금액(TOTAL_ORD_AMT) * 100.00
    -- T_ORD 테이블을 불필요하게 반복 접근하는 쿼리
    -- 성능에 문제도 있지만, SQL을 변경할 때 손이 많이 가서 번거로움
    -- 총 주문금액의 기준이 바뀌면, 두 서브쿼리를 모두 변경해야 함
    select 	TO_CHAR(T1.ORD_DT, 'YYYYMMDD') as ORD_YMD,
    		SUM(T1.ORD_AMT) as ORD_AMT,
    		(
    			select	SUM(A.ORD_AMT)
    			from	T_ORD A
    			where	A.ORD_DT >= DATE '2017-08-01'
    			and		A.ORD_DT < DATE '2017-09-01'
    		) as TOTAL_ORD_AMT,
    		ROUND(
    			SUM(T1.ORD_AMT) / (
    				select	SUM(A.ORD_AMT)
    				from	T_ORD A
    				where 	A.ORD_DT >= DATE '2017-08-01'
    				and		A.ORD_DT < DATE '2017-09-01'
    			) * 100, 2
    		) as ORD_AMT_RT
    from	T_ORD T1
    where	T1.ORD_DT >= DATE '2017-08-01'
    and		T1.ORD_DT < DATE '2017-09-01'
    group by TO_CHAR(T1.ORD_DT, 'YYYYMMDD');
    
    -- ************************************************
    -- PART I - 4.1.2 SQL3
    -- ************************************************
    
    -- 인라인-뷰를 사용해 반복 서브쿼리를 제거하는 방법
    -- 같은 서브쿼리를 반복해서 사용해야 한다면 인라인ㅡ뷰를 고민하기
    select	T1.ORD_YMD,
    		T1.ORD_AMT,
    		T1.TOTAL_ORD_AMT,
    		ROUND(T1.ORD_AMT / T1.TOTAL_ORD_AMT * 100, 2) as ORD_AMT_RT
    from	(
    	select	TO_CHAR(T1.ORD_DT, 'YYYYMMDD') as ORD_YMD,
    			SUM(T1.ORD_AMT) as ORD_AMT,
    			(
    				select	SUM(A.ORD_AMT)
    				from	T_ORD A
    				where	A.ORD_DT >= DATE '2017-08-01'
    				and		A.ORD_DT < DATE '2017-09-01'
    			) as TOTAL_ORD_AMT
    	from	T_ORD T1
    	where	T1.ORD_DT >= DATE '2017-08-01'
    	and		T1.ORD_DT < DATE '2017-09-01'
    	group by TO_CHAR(T1.ORD_DT, 'YYYYMMDD')
    ) T1;
    
    -- ************************************************
    -- PART I - 4.1.2 SQL4
    -- ************************************************
    
    -- 카테시안-조인을 사용해 반복 서브쿼리를 제거하는 방법
    select	TO_CHAR(T1.ORD_DT, 'YYYYMMDD') as ORD_YMD
    		,SUM(T1.ORD_AMT) as ORD_AMT
    		,MAX(T2.TOTAL_ORD_AMT) as TOTAL_ORD_AMT
    		,ROUND(SUM(T1.ORD_AMT) / MAX(T2.TOTAL_ORD_AMT) * 100, 2) as ORD_AMT_RT
    from	T_ORD T1
    		,(	select	SUM(A.ORD_AMT) as TOTAL_ORD_AMT
    			from	T_ORD A
    			where	A.ORD_DT >= TO_TIMESTAMP('2017-08-01', 'YYYY-MM-DD')
    			and		A.ORD_DT < TO_TIMESTAMP('2017-09-01', 'YYYY-MM-DD')
    		) T2
    where	T1.ORD_DT >= TO_TIMESTAMP('2017-08-01', 'YYYY-MM-DD')
    and		T1.ORD_DT < TO_TIMESTAMP('2017-09-01', 'YYYY-MM-DD')
    group by TO_CHAR(T1.ORD_DT, 'YYYYMMDD');
    
    -- SELECT 절의 단독 서브쿼리
    -- 인라인ㅡ뷰
    -- 카테시안ㅡ조인
    
    -- 서브쿼리를 제거하면 SQL 성능이 좋아질 수 있음
    
    -- 비슷하면서 약간씩 다른 서브쿼리를 매우 많이 사용하는 SQL
    -- 인덱스나 힌트로는 성능 개선에 한계가 있음
    -- SQL 전체를 뜯어고쳐서 반복되는 서브쿼리를 제거해야 함
    ```
    
    ### 4.1.3 SELECT 절의 상관 서브쿼리
    
    메인 SQL에서 값을 받아 처리한다.
    
    대부분의 조인을 해결할 수 있다.
    
    적절하게 사용하기
    
    ```sql
    -- ************************************************
    -- PART I - 4.1.3 SQL1
    -- ************************************************
    
    -- 코드값을 가져오는 SELECT 절 상관 서브쿼리
    -- 코드명 처리는 조인보다는 SELECT 절의 상관쿼리를 사용하는 것이 일반적
    -- 코드처럼 값의 종류가 많지 않은 경우는 캐싱 효과로 성능이 더 좋아질 수도 있음
    select	T1.ITM_TP
    		,(select	A.BAS_CD_NM
    		  from		C_BAS_CD A -- 기준코드 테이블
    		  where 	A.BAS_CD_DV = 'ITM_TP'
    		  	and		A.BAS_CD = T1.ITM_TP
    		  	and		A.LNG_CD = 'KO'
    		) as ITM_TP_NM -- 메인 SQL의 값을 가져와 사용
    		,T1.ITM_ID
    		,T1.ITM_NM
    from	M_ITM T1;
    
    -- ************************************************
    -- PART I - 4.1.3 SQL2
    -- ************************************************
    
    -- 고객정보를 가져오는 SELECT 절 상관 서브쿼리
    -- 조인으로 해결하는 것이 좋은 SQL
    select	T1.CUS_ID
    		,TO_CHAR(T1.ORD_DT, 'YYYYMMDD') as ORD_YMD
    		-- 불필요하게 M_CUS_를 두 번이나 접근
    		-- 성능에서 손해 볼 가능성이 큼
    		,(select A.CUS_NM from M_CUS A where A.CUS_ID = T1.CUS_ID limit 1) as CUS_NM
    		,(select A.CUS_GD from M_CUS A where A.CUS_ID = T1.CUS_ID limit 1) as CUS_GD
    		,T1.ORD_AMT
    from 	T_ORD T1
    where	T1.ORD_DT >= TO_DATE('2017-08-01', 'YYYY-MM-DD')
    and		T1.ORD_DT < TO_DATE('2017-09-01', 'YYYY-MM-DD');
    
    -- ************************************************
    -- PART I - 4.1.3 SQL3
    -- ************************************************
    
    -- 인라인-뷰 안에서 SELECT 절 서브쿼리를 사용한 예
    -- SELECT 절의 서브쿼리는 가장 바깥의 SELECT 절에만 사용하도록 노력해야 함
    select	T1.CUS_ID
    		,SUBSTRING(T1.ORD_YMD from 1 for 6) as ORD_YM
    		,MAX(T1.CUS_NM) as CUS_NM
    		,MAX(T1.CUS_GD) as CUS_GD
    		,T1.ORD_ST_NM
    		,T1.PAY_TP_NM
    		,SUM(T1.ORD_AMT) as ORD_AMT
    from	(
    			select	T1.CUS_ID
    					,TO_CHAR(T1.ORD_DT, 'YYYYMMDD') as ORD_YMD
    					,T2.CUS_NM
    					,T2.CUS_GD
    					,(select A.BAS_CD_NM
    					  from		C_BAS_CD A
    					  where		A.BAS_CD_DV = 'ORD_ST'
    					  and		A.BAS_CD = T1.ORD_ST
    					  and		A.LNG_CD = 'KO'
    					  limit 1) as ORD_ST_NM
    					,(select A.BAS_CD_NM
    					  from		C_BAS_CD A
    					  where		A.BAS_CD_DV = 'PAY_TP'
    					  and		A.BAS_CD = T1.PAY_TP
    					  and		A.LNG_CD = 'KO'
    					  limit 1) as PAY_TP_NM
    					 ,T1.ORD_AMT
    			from	T_ORD T1
    					,M_CUS T2
    			where	T1.ORD_DT >= TO_DATE('2017-08-01', 'YYYY-MM-DD')
    			and		T1.ORD_DT < TO_DATE('2017-09-01', 'YYYY-MM-DD')
    			and		T1.CUS_ID = T2.CUS_ID
    		) T1
    group by T1.CUS_ID
    		,SUBSTRING(T1.ORD_YMD from 1 for 6)
    		,T1.ORD_ST_NM
    		,T1.PAY_TP_NM;
    
    -- ************************************************
    -- PART I - 4.1.3 SQL4
    -- ************************************************
    
    -- 서브쿼리 안에서 조인을 사용한 예
    -- 기본적으로 상관 서브쿼리는 메인 SQL의 결과 건수만큼 '반복' 수행됨
    -- '과유불급'
    select	T1.ORD_DT
    		,T2.ORD_QTY
    		,T2.ITM_ID
    		,T3.ITM_NM
    		,(	select	SUM(B.EVL_PT) / COUNT(*)
    			from	M_ITM A
    					,T_ITM_EVL B
    			where	A.ITM_TP = T3.ITM_TP
    			and		B.ITM_ID = A.ITM_ID
    			and		B.EVL_DT < T1.ORD_DT
    		  ) as ITM_TP_EVL_PT_AVG
    from	T_ORD T1
    		,T_ORD_DET T2
    		,M_ITM T3
    where	T1.ORD_DT >= TO_DATE('2017-08-01', 'YYYY-MM-DD')
    and		T1.ORD_DT < TO_DATE('2017-09-01', 'YYYY-MM-DD')
    and		T3.ITM_ID = T2.itm_id
    and		T1.ORD_SEQ = T2.ord_seq
    order by T1.ORD_DT, T2.ITM_ID;
    ```
    
    ### 4.1.4 SELECT 절 서브쿼리 - 단일 값
    
    SELECT 절의 서브쿼리가 두 건 이상의 결과를 내보내거나 두 개 이 컬럼 이상의 결과를 내보내면 안 된다.
    
    ```sql
    -- ************************************************
    -- PART I - 4.1.4 SQL1
    -- ************************************************
    
    -- 실행이 불가능한 SELECT 절의 서브쿼리
    --SELECT 절의 서브쿼리에서 두 컬럼을 지정.
    select	T1.ORD_DT
    		,T1.CUS_ID
    		,(select	A.CUS_NM ,A.CUS_GD
    		  from		M_CUS A
    		  where		A.CUS_ID = T1.CUS_ID
    		  )	as CUS_NM_GC
    from	T_ORD T1
    where	T1.ORD_DT >= TO_DATE('2017-04-01', 'YYYY-MM-DD')
    and		T1.ORD_DT < TO_DATE('2017-05-01', 'YYYY-MM-DD');
    
    --SELECT 절의 서브쿼리에서 두 건 이상의 데이터가 나오는 경우.
    select	T1.ORD_DT
    		,T1.CUS_ID
    		,(select	A.ITM_ID
    		  from		T_ORD_DET A
    		  where		A.ORD_SEQ = T1.ORD_SEQ
    		  ) as ITM_LIST
    from	T_ORD T1
    where	T1.ORD_DT >= TO_DATE('2017-04-01', 'YYYY-MM-DD')
    and		T1.ORD_DT < TO_DATE('2017-05-01', 'YYYY-MM-DD');
    
    -- ************************************************
    -- PART I - 4.1.4 SQL2
    -- ************************************************
    
    -- 고객 이름과 등급을 합쳐서 하나의 컬럼으로 처리
    -- 단가(UNT_PRC)와 주문수량(ORD_QTY)를 곱해서 주문금액으로 처리.
    select	T1.ORD_DT
    		,T1.CUS_ID
    		,(select	A.CUS_NM || '(' || A.CUS_GD || ')'
    		  from		M_CUS A
    		  where 	A.CUS_ID = T1.CUS_ID
    		  ) as CUS_NM_GD
    		,(select	SUM(A.UNT_PRC * A.ORD_QTY)
    		  from		T_ORD_DET A
    		  where		A.ORD_SEQ = T1.ORD_SEQ
    		  ) as ORD_AMT
    from	T_ORD T1
    where	T1.ORD_DT >= TO_DATE('2017-04-01', 'YYYY-MM-DD')
    and		T1.ORD_DT < TO_DATE('2017-05-01', 'YYYY-MM-DD');
    
    -- ************************************************
    -- PART I - 4.1.4 SQL3
    -- ************************************************
    
    -- 고객별 마지막 ORD_SEQ의 주문금액
    select	T1.CUS_ID
    		,T1.CUS_NM
    		,(select	cast(SUBSTRING(MAX(
    								   LPAD(cast(A.ORD_SEQ as TEXT), 8, '0') || CAST(A.ORD_AMT as TEXT)
    							   ), 9) as numeric)
    		  from		T_ORD A
    		  where		A.CUS_ID = T1.CUS_ID
    		) as LAST_ORD_AMT
    from	M_CUS T1
    order by T1.CUS_ID;
    
    -- ************************************************
    -- PART I - 4.1.4 SQL4
    -- ************************************************
    
    -- 고객별 마지막 ORD_SEQ의 주문금액 – 중첩된 서브쿼리
    -- SELECT 절의 서브쿼리는 조회되는 데이터 건수만큼 반복 실행됨
    -- 조회되는 결과 건수가 작을 때만 사용
    select	T1.CUS_ID
    		,T1.CUS_NM
    		,(
    			select	B.ORD_AMT
    			from 	T_ORD B
    			where	B.ORD_SEQ =
    						(select MAX(A.ORD_SEQ) from T_ORD A where A.CUS_ID = T1.CUS_ID)
    		) as LAST_ORD_AMT
    from 	M_CUS T1
    order by T1.CUS_ID;
    
    -- ************************************************
    -- PART I - 4.1.4 SQL5
    -- ************************************************
    
    -- 잠재적인 오류가 존재하는 서브쿼리 – 정상 실행
    select	T1.ORD_DT
    		,T1.CUS_ID
    		,(select	A.ORD_QTY
    		  from		T_ORD_DET A
    		  where 	A.ORD_sEQ = T1.ORD_SEQ
    		  limit 1) as ORD_AMT
    from	T_ORD T1
    where	T1.ORD_SEQ = 2015;
    
    -- ************************************************
    -- PART I - 4.1.4 SQL6
    -- ************************************************
    
    -- 잠재적인 오류가 존재하는 서브쿼리 – 오류 발생
    --1. 오류가 발생하는 서브쿼리(ORD_SEQ =2015)
    select	T1.ORD_DT
    		,T1.CUS_ID
    		,(select	A.ORD_QTY
    		  from		T_ORD_DET A
    		  where		A.ORD_SEQ = T1.ORD_SEQ
    		  ) as ORD_AMT
    from	T_ORD T1
    where 	T1.ORD_SEQ = 2015;
    
    --2. T_ORD_DET에 ORD_SEQ가 2015인 데이터는 두 건이 존재한다.
    select	T1.*
    from	T_ORD_DET T1
    where 	T1.ORD_SEQ = 2015;
    ```
    
    ### 4.1.5 WHERE 절 단독 서브쿼리
    
    ```sql
    -- ************************************************
    -- PART I - 4.1.5 SQL1
    -- ************************************************
    
    -- 마지막 주문 한 건을 조회하는 SQL, ORD_SEQ가 가장 큰 데이터가 마지막 주문이다.
    select	*
    from	T_ORD T1
    where	T1.ORD_SEQ = (select MAX(A.ORD_SEQ) from T_ORD A);
    
    -- ************************************************
    -- PART I - 4.1.5 SQL2
    -- ************************************************
    
    -- 마지막 주문 한 건을 조회하는 SQL, ORDER BY와 ROWNUM을 사용
    -- LIMIT : 단순한 행 제한
    -- ROW_NUMBER() : 복잡한 필터링이나 조건에 따른 순위
    -- ORD_SEQ에 대한 인덱스 필수
    select	*
    from	T_ORD T1
    order by T1.ORD_SEQ desc
    limit 1;
    
    with RankedOrders as (
    	select T1.*, ROW_NUMBER() over (order by T1.ORD_SEQ DESC) as ROW_NUM
    	from T_ORD T1
    )
    select *
    from RankedOrders
    where ROW_NUM = 1;
    
    -- ************************************************
    -- PART I - 4.1.5 SQL3
    -- ************************************************
    
    -- 마지막 주문 일자의 데이터를 가져오는 SQL
    select	*
    from	T_ORD T1
    where 	T1.ORD_DT = (select MAX(A.ORD_DT) from T_ORD A);
    
    -- ************************************************
    -- PART I - 4.1.5 SQL4
    -- ************************************************
    
    -- 3월 주문 건수가 4건 이상인 고객의 3월달 주문 리스트
    select	*
    from	T_ORD T1
    where	T1.ORD_DT >= '2017-03-01'::DATE
    and 	T1.ORD_DT < '2017-04-01'::DATE
    and 	T1.CUS_ID in (
    			select	A.CUS_ID
    			from 	T_ORD A
    			where 	A.ORD_DT >= '2017-03-01'::DATE
    			and 	A.ORD_DT < '2017-04-01'::DATE
    			group by A.cus_id
    			having COUNT(*) >= 4
    			);
    
    -- ************************************************
    -- PART I - 4.1.5 SQL5
    -- ************************************************
    
    -- 3월 주문 건수가 4건 이상인 고객의 3월달 주문 리스트 – 조인으로 처리
    select 	T1.*
    from	T_ORD T1
    join	(
    			select	A.CUS_ID
    			from	T_ORD A
    			where 	A.ORD_DT >= '2017-03-01'::DATE
    			and 	A.ORD_DT < '2017-04-01'::DATE
    			group by A.cus_id
    			having COUNT(*) >= 4
    		) T2
    on 		T1.CUS_ID = T2.cus_id
    where	T1.ORD_DT >= '2017-03-01'::DATE
    and		T1.ORD_DT < '2017-04-01'::DATE;
    ```
    
    ### 4.1.6 WHERE 절 상관 서브쿼리
    
    데이터의 존재 여부를 파악할 때 자주 사용한다.
    
    특정 일자나 특정 월에 주문이 존재하는 고객 리스트를 뽑을 때 유용하다.
    
    ```sql
    -- ************************************************
    -- PART I - 4.1.6 SQL1
    -- ************************************************
    
    -- 3월에 주문이 존재하는 고객들을 조회
    -- 반대는 NOT EXISTS
    -- 다른 테이블에 데이터 존재 여부를 파악할 때 유용
    select 	*
    from 	M_CUS T1
    where 	EXISTS(
    			  select	1
    			  from		T_ORD A
    			  where		A.CUS_ID = T1.cus_id
    			  and		A.ORD_DT >= '2017-03-01'::DATE
    			  and 		A.ORD_DT < '2017-04-01'::DATE
    			  );
    
    -- ************************************************
    -- PART I - 4.1.6 SQL2
    -- ************************************************
    
    -- 3월에 ELEC 아이템유형의 주문이 존재하는 고객들을 조회
    select	*
    from	M_CUS T1
    where 	EXISTS(
    			select	1
    			from	T_ORD A
    			join	T_ORD_DET B on A.ORD_SEQ = B.ord_seq
    			join	M_ITM C on B.ITM_ID = C.itm_id 
    			and		A.ORD_DT >= '2017-03-01'::DATE
    			and 	A.ORD_DT < '2017-04-01'::DATE
    			and 	C.ITM_TP = 'ELEC'
    			);
    
    -- ************************************************
    -- PART I - 4.1.6 SQL3
    -- ************************************************
    
    -- 전체 고객을 조회, 3월에 주문이 존재하는지 여부를 같이 보여줌
    select	T1.CUS_ID, T1.CUS_NM,
    		(case when exists(
    				select	1
    				from	T_ORD A
    				where	A.CUS_ID = T1.CUS_ID
    				and		A.ORD_DT >= '2017-03-01'::DATE
    				and		A.ORD_DT < '2017-04-01'::DATE
    			)
    			then 'Y'
    			else 'N'
    		end) as ORD_YN_03
    from	M_CUS T1;
    ```
    
- 4.2 MERGE
    
    ### 4.2.1 MERGE
    
    MERGE는 한 문장으로 INSERT와 UPDATE를 동시에 처리할 수 있다.
    
    한 건의 데이터는 INSERT와 UPDATE 중 하나만이 수행된다.
    
    MERGE 대상이 이미 존재하면 UPDATE를, 대상이 존재하지 않으면 INSERT를 수행하는 방식이다.
    
    ```sql
    -- ************************************************
    -- PART I - 4.2.1 SQL1
    -- ************************************************
    
    -- MERGE 문을 위한 테스트 테이블 생성
    -- 'CREATE TABLE  AS'(줄여서 CTAS)
    -- CTAS로는 테이블의 PK까지는 생성되지 않음
    create table M_CUS_CUD_TEST as
    select	*
    from	M_CUS T1;
    
    alter table M_CUS_CUD_TEST
    	add constraint PK_M_CUS_CUD_TEST primary key (CUS_ID);
    
    -- ************************************************
    -- PART I - 4.2.1 SQL2
    -- ************************************************
    
    -- CUS_0090 고객을 입력하거나 변경하는 PL/SQL
    do $$
    declare
    	v_EXISTS_YN VARCHAR(1);
    begin
    	-- 고객ID가 이미 있는지 확인하는 SQL
    	select COALESCE(MAX('Y'), 'N')
    	into v_EXISTS_YN
    	from M_CUS_CUD_TEST T1
    	where T1.CUS_ID = 'CUS_0090';
    
    	-- Conditional Logic for inserting or updating
    	if v_EXISTS_YN = 'N' then
    		-- Insert new customer
    		insert into M_CUS_CUD_TEST (CUS_ID, CUS_NM, CUS_GD)
    		values ('CUS_0090', 'NAME_0090', 'A');
    	
    		raise notice 'INSERT NEW CUST';
    	else
    		-- Update existing customer
    		update M_CUS_CUD_TEST T1
    		set T1.CUS_NM = 'NAME_0090',
    			T1.CUS_GD = 'A'
    		where T1.CUS_ID = 'CUS_0090';
    		
    		raise notice 'UPDATE OLD CUST';
    	end if;
    		
    	commit;
    end $$;
    
    -- ************************************************
    -- PART I - 4.2.1 SQL3
    -- ************************************************
    
    -- 고객을 입력하거나 변경하는 SQL – MERGE 문으로 처리
    -- INSERT와 UPDATE를 나누어 개발하는 것이 명확한 경우가 더 많음
    merge into M_CUS_CUD_TEST T1 -- MERGE 대상
    using (
    	select 	'CUS_0090' as CUS_ID,
    			'NAME_0090' as CUS_NM,
    			'A' as CUS_GD
    ) T2 -- 비교대상 (실제 테이블도 사용 가능)
    on (T1.CUS_ID = T2.CUS_ID) -- 비교 조건
    when matched then
    	update set 	CUS_NM = T2.CUS_NM,
    				CUS_GD = T2.CUS_GD
    when not matched then
    	insert (CUS_ID, CUS_NM, CUS_GD)
    	values (T2.CUS_ID, T2.CUS_NM, T2.CUS_GD);
    ```
    
    ### 4.2.2 MERGE를 사용한 UPDATE
    
    MERGE  문장에서 WHEN MATCHED THEN 절만 사용하면, 해당 MERGE 문은 UPDATE만 처리한다.
    
    ```sql
    -- ************************************************
    -- PART I - 4.2.2 SQL1
    -- ************************************************
    
    -- 월별고객주문 테이블 생성 및 기조 데이터 입력
    -- 테이블 생성
    create table S_CUS_YM
    (
    	BAS_YM	VARCHAR(6) not null,
    	CUS_ID	VARCHAR(40) not null,
    	ITM_TP	VARCHAR(40) not null,
    	ORD_QTY	NUMERIC(18,3) null,
    	ORD_AMT	NUMERIC(18,3) null,
    	primary key (BAS_YM, CUS_ID, ITM_TP)
    );
    
    -- 데이터 삽입
    insert into S_CUS_YM (BAS_YM, CUS_ID, ITM_TP, ORD_QTY, ORD_AMT)
    select	'201702' as BAS_YM,
    		T1.CUS_ID,
    		T2.BAS_CD as ITM_TP,
    		null as ORD_QTY,
    		null as ORD_AMT
    from	M_CUS T1,
    		C_BAS_CD T2
    where 	T2.BAS_CD_DV = 'ITM_TP'
    and		T2.LNG_CD = 'KO';
    
    -- 커밋
    commit;
    
    -- ************************************************
    -- PART I - 4.2.2 SQL2
    -- ************************************************
    
    -- 월별고객주문의 주문수량, 주문금액 업데이트
    update S_CUS_YM T1
    set 
    	ORD_QTY = (
    		select SUM(B.ORD_QTY)
    		from T_ORD A
    			join T_ORD_DET B on A.ORD_SEQ = B.ORD_SEQ
    			join M_ITM C on C.ITM_ID = B.ITM_ID
    		where C.ITM_TP = T1.ITM_TP
    		and A.CUS_ID = T1.CUS_ID
    		and A.ORD_DT >= TO_DATE(T1.BAS_YM || '01', 'YYYYMMDD')
    		and A.ORD_DT < (TO_DATE(T1.BAS_YM || '01', 'YYYYMMDD') + interval '1 month')
    	),
    	ORD_AMT = (
    		select SUM(B.UNT_PRC * B.ORD_QTY)
    		from T_ORD A
    			join T_ORD_DET B on A.ORD_sEQ = B.ORD_SEQ
    			join M_ITM C on C.ITM_ID = B.ITM_ID
    		where C.ITM_TP = T1.ITM_TP
    		and A.CUS_ID = T1.CUS_ID
    		and A.ORD_DT >= TO_DATE(T1.BAS_YM || '01', 'YYYYMMDD')
    		and A.ORD_DT < (TO_DATE(T1.BAS_YM || '01', 'YYYYMMDD') + interval '1 month')
    	)
    where T1.BAS_YM = '201702';
    
    -- ************************************************
    -- PART I - 4.2.2 SQL3
    -- ************************************************
    
    -- 월별고객주문의 주문금액, 주문수량 업데이트 – 머지 사용
    merge into S_CUS_YM T1
    using ( -- 서브쿼리
    	select	A.CUS_ID,
    			C.ITM_TP,
    			SUM(B.ORD_QTY) as ORD_QTY,
    			SUM(B.UNT_PRC * B.ORD_QTY) as ORD_AMT
    	from	T_ORD A
    			join T_ORD_DET B on A.ORD_SEQ = B.ORD_SEQ
    			join M_ITM C on C.ITM_ID = B.ITM_ID
    	where	A.ORD_DT >= TO_DATE('201702' || '01', 'YYYYMMDD')
    	and		A.ORD_DT < (TO_DATE('201702' || '01', 'YYYYMMDD') + interval '1 month')
    	group by A.CUS_ID, C.ITM_TP
    ) T2
    on (T1.BAS_YM = '201702'
    	and T1.CUS_ID = T2.CUS_ID
    	and T1.ITM_TP = T2.ITM_TP)
    when matched then
    	update set	ORD_QTY = T2.ORD_QTY,
    				ORD_AMT = T2.ORD_AMT;
    
    -- ************************************************
    -- PART I - 4.2.2 SQL4
    -- ************************************************
    
    -- 월별고객주문의 주문금액, 주문수량 업데이트 – 반복 서브쿼리 제거
    update S_CUS_YM T1
    set ORD_QTY = subquery.ORD_QTY,
    	ORD_AMT = subquery.ORD_AMT
    from (
    	select	A.CUS_ID,
    			C.ITM_TP,
    			SUM(B.ORD_QTY) as ORD_QTY,
    			SUM(B.UNT_PRC * B.ORD_QTY) as ORD_AMT
    	from T_ORD A
    	join T_ORD_DET B on A.ORD_SEQ = B.ORD_SEQ
    	join M_ITM C on C.ITM_ID = B.itm_id
    	where A.ORD_DT >= TO_DATE('201702' || '01', 'YYYYMMDD')
    	  and A.ORD_DT < (TO_DATE('201702' || '01', 'YYYYMMDD') + interval '1 month')
    	group by A.CUS_ID, C.ITM_TP
    ) as subquery
    where T1.BAS_YM = '201702'
      and T1.CUS_ID = subquery.cus_id
      and T1.ITM_TP = subquery.ITM_TP;
    ```
    
- 4.3 WITH
    
    ### 4.3.1 WITH
    
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
