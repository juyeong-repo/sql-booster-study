📌오라클 → PostgreSql 15로 변환

- 7.3 MERGE 조인과 성능
    
    ### 7.3.1 대량의 데이터 처리
    
    머지 조인은 대량의 데이터를 조인할 때 적합하다.
    
    ### Nested Loop만 사용
    
    ```sql
    set enable_hashjoin to off;
    set enable_mergejoin to off;
    ```
    
    ### Merge Join만 사용
    
    ```sql
    set enable_hashjoin to off;
    set enable_nestloop to off;
    ```
    
    ```sql
    -- ************************************************
    -- PART II - 7.3.1 SQL
    -- ************************************************
    
       -- 고객별 2월 전체 주문금액 
       explain analyz
       select 
          T1.CUS_ID,
          MAX(T1.CUS_NM) as CUST_NM,
          MAX(T1.CUS_GD) as CUS_GD,
          COUNT(*) as ORD_CNT,
          SUM(T2.ORD_AMT) as ORD_AMT,
          SUM(SUM(T2.ORD_AMT)) over () as TTL_ORD_AMT
       from
          M_CUS T1
       join
          T_ORD_BIG T2 on T1.CUS_ID = T2.CUS_ID
       where
          T2.ORD_YMD like '201702%'
       group by
          T1.CUS_ID;
    ```
    
    ### ✅ MERGE 조인 실행 순서
    
    1. `T_ORD_BIG`: 병렬 Full Scan 후 `ORD_YMD LIKE '201702%'` 필터
    2. `T_ORD_BIG`: CUS_ID 기준 정렬 (비용 큼)
    3. `M_CUS`: Seq Scan 후 CUS_ID 기준 정렬 (비용 적음)
    4. 두 테이블을 Merge Join으로 조인 (정렬된 CUS_ID 기준)
    5. GroupAggregate + Window Function 처리
    
    ### ✅ NL 조인 실행 순서
    
    1. `T_ORD_BIG`: 병렬 Full Scan → `ORD_YMD LIKE '201702%'` 필터링
    2. 각 `CUS_ID`에 대해:
        - `PK_M_CUS` 인덱스로 고객 정보 조회 (Index Scan + Table Access)
        - Memoize로 중복 조회 최소화
    3. Nested Loop Join으로 조인 결과 생성
    4. GroupAggregate + Window Function 처리
    
    ### 7.3.2 필요한 인덱스
    
    머지 조인은 조인에 참여하는 데이터를 각각 조회해서 조인을 처리한다.
    
    ∴ 조인에 참여하는 테이블별로 대상을 줄일 수 있는 조건에 인덱스를 만들어 주면 된다.
    
    단, 해당 조건에 인덱스를 사용하는 것이 ‘TABLE ACCESS FULL’ 보다는 좋은 성능을 낼 수 있어야 한다.
    
    ```sql
    SET enable_seqscan = off;  -- 인덱스 강제 사용 유도
    SET enable_seqscan = on;
    ```
    
- 7.4 HASH 조인과 성능
    
    ### 7.4.1 대량의 데이터 처리
    
    NL 조인은 많은 양의 데이터를 조인 처리하기에는 적합하지 않다.
    
    ∴ 많은 양의 데이터를 조인하려면 머지 조인을 사용해야 한다.
    
    하지만 머지 조인도 데이터를 정렬해야 하는 부담이 있다.
    
    ∴ 해시 조인으로 단점을 해결할 수 있다.
    
    해시 조인은 해시 함수를 사용하기 때문에 CPU와 메모리에 추가적인 부하가 발생한다.
    
    하지만 머지 조인에 비하면 월등한 성능을 가지고 있기 때문에 대용량 데이터의 조인은 해시 조인이 필수다.
    
    ```sql
    -- ************************************************
    -- PART II - 7.4.1 SQL1
    -- ************************************************
    
    	-- T_ORD_BIG 전체를 조인 – 머지 조인으로 처리
    	set enable_hashjoin = off;
    	set enable_mergejoin = on;
    	set enable_nestloop = off;
    	
    	explain analyze
    	select
    		T1.CUS_ID,
    		MAX(T1.CUS_NM) as CUS_NM,
    		MAX(T1.CUS_GD) as CUS_GD,
    		COUNT(*) as ORD_CNT,
    		SUM(T2.ORD_AMT) as ORD_AMT,
    		SUM(SUM(T2.ORD_AMT)) over () as TTL_ORD_AMT
    	from M_CUS T1
    	join T_ORD_BIG T2 on T1.CUS_ID = T2.CUS_ID
    	group by T1.CUS_ID;
    
    -- ************************************************
    -- PART II - 7.4.1 SQL2
    -- ************************************************
    
    	-- T_ORD_BIG 전체를 조인 – 해시 조인으로 처리
    	set enable_hashjoin = on;
    	set enable_mergejoin = off;
    	set enable_nestloop = off;
    	
    	explain analyze
    	select
    		T1.CUS_ID,
    		MAX(T1.CUS_NM) as CUS_NM,
    		MAX(T1.CUS_GD) as CUS_GD,
    		COUNT(*) as ORD_CNT,
    		SUM(T2.ORD_AMT) as ORD_AMT,
    		SUM(SUM(T2.ORD_AMT)) over () as TTL_ORD_AMT
    	from M_CUS T1
    	join T_ORD_BIG T2 on T1.CUS_ID = T2.CUS_ID
    	group by T1.CUS_ID;
    ```
    
    ### 🧾 요약 비교
    
    | 항목 | MERGE JOIN 실행계획 | HASH JOIN 실행계획 |
    | --- | --- | --- |
    | **조인 방식** | `Merge Join` | `Hash Join` |
    | **T_ORD_BIG 접근 방식** | `Parallel Index Scan (x_t_ord_big_4)` | `Parallel Seq Scan` |
    | **실행 시간** | **118,609ms (약 2분)** | **5,537ms (약 5.5초)** |
    | **총 조인 결과 건수** | 약 6,720,000건 (×3 workers) | 동일 |
    | **병렬 처리 효과** | 있음 (정렬 비용 큼) | 있음 (Hash Join 효율적) |
    
    ### 7.4.2 빌드 입력 선택의 중요성
    
    성능 향상을 위해 선행 집합 선택이 중요하다.
    
    해시 조인의 경우, 선행 집합은 빌드 입력(Build-Input)으로 처리하며, 후행 집합은 검증 입력(Probe-Input)으로 처리된다.
    
    해시 조인은 빌드 입력의 데이터가 적으면 적을수록 성능에 유리하다.
    
    대용량 테이블 간에 조인이 수행되는 경우는 어느 쪽을 빌드 입력으로 선택하든 성능에 큰 차이가 나지 않을 수 있지만 습관적으로 족므이라도 작은 쪽을 선행 집합으로 해시 조인을 처리하는 것이 좋다.
    
    ### 7.4.3 대량의 데이터에만 사용할 것인가?
    
    해시 조인은 메모리와 CPU를 비교적 많이 사용한다는 점과 대용량 데이터 조인에 적합하다는 이유로 소량의 데이터를 무조건 NL 조인으로 변경할 필요는 없다.
    
    특정 SQL이 매우 많이 사용되면서 CPU 접유 시간이 높은 상황이 아니라면 굳이 해시 조인을 제거하려고 노력할 필요는 없다.
    
    ### 7.4.4 어떤 조인을 사용할 것인가?
    
    OLTP 환경의 로그인 처리, 계좌이체, 주문처리 같은 자주 실행되는 SQL은 NL 조인만으로 처리하는 것이 데이터베이스 전체 성능에 도움이 된다.
    
    단, NL 조인의 성능이 확보되도록 적절한 인덱스가 구성되어 있어야 한다.
    
    대량의 데이터를 조회해서 분석을 수행해야 한다면 해시 조인이 유용하다.
    
    조인 조건 컬럼에 적절한 인덱스를 만들기 어려울 때도 해시 조인을 활용한다.
    
    힌트를 사용하지 않는 한 어떤 조인을 할지는 옵티마이져가 결정한다.
    
    옵티마이져가 제일 나은 선택을 할 수 있게 인덱스를 잘 구성해주는 것이 중요하다.
    
    인덱스를 잘 구성하려면 SQL이 어떤 조인 방식으로 처리하는 것이 좋은지 먼저 판단할 수 있어야 한다.
    
    꾸준한 연습 필요
