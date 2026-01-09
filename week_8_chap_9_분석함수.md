분석함수는 조회된 데이터를 가지고 다양한 분석을 손쉽게 구현할 수 있게 하는 함수이다

보통 OVER절과 함께 사용되고, OVER절 안에 ‘PARTITION BY’, ‘ORDER BY’등을 제공한다

# OVER 절

### **PARTITION BY**

- OVER절의 괄호 안에 사용하는 구문
- 여러 컬럼 지정 가능
- SELECT 절에서 분석함수별로 다르게 지정할 수도 있음

```sql
SELECT T1.CUS_ID, TO_CHAR(T1.ORD_DT,'YYYYMM') ORD_YM, T1.ORD_ST
			 ,SUM(T1.ORD_AMT) ORD_AMT
			 ,SUM(SUM(T1.ORD_AMT)) OVER (PARTITION BY T1.CUS_DT) BY_CUST_AMT
			 ,SUM(SUM(T1.ORD_AMT)) OVER (PARTITION BY T1.ORD_ST) BY_ORD_ST_AMT
			 ,SUM(SUM(T1.ORD_AMT)) OVER (PARTITION BY T1.CUS_ID, TO_CHAR(T1.ORD_DT,'YYYMM')) BY_CUST_YM_AMT
FROM   T_ORD T1
WHERE  T1.CUS_ID IN ('CUS_0002', 'CUS_0003')
AND    T1.ORD_DT >= TO_DATE('20170301','YYYYMMDD')
AND    T1.ORD_DT < TO_DATE('20170601','YYYYMMDD')
GROUP BY T1.CUS_ID, TO_CHAR(T1.ORD_DT,'YYYYMM'), T1.ORD_ST
ORDER BY T1.CUS_ID, TO_CHAR(T1.ORD_DT,'YYYYMM'), T1.ORD_ST;
```

### **ORDER BY**

- SELECT 문에서는 조회된 결과를 정렬
- OVER절 안에서는 각 로우별로 ORDER BY에 따라 분석 대상이 다르게 정해진다
    - 현재 데이터까지의 누적 합계를 구할 때 유용함

## 기타 분석함수

- RANK, DENSE_RANK
    - 순위 분석함수
- ROW_NUMBER
    - 중복된 순위를 내보내지 않음
- LAG, LEAD
    - LAG - 자신의 이전 값을 가져옴
    - LEAD - 자신의 이후 값을 가져옴
    - 자신의 몇 건 이전/이후 값을 가져올지 결정할 수 있음
- 

```sql
오라클
- LAG(컬럼명, offset) OVER ([PARTITION BY~] ORDER BY ~)
- LEAD(컬럼명, offset) OVER ([PARTITION BY~] ORDER BY ~)

MySQL도 8.0부터 지원함
SELECT user_id, created,
  LAG(created, 1) OVER (PARTITION BY user_id ORDER BY created) AS prev_created,
  LEAD(created 1) OVER (PARTITION BY user_id ORDER BY created) AS next_created
FROM user_logs;

```
