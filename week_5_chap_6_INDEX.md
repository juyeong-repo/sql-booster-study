🌟 인덱스는 SQL 성능 개선을 위한 가장 기본적이면서 치명적인 무기

SQL 성능을 최대로 끌어올리기 위해서는 최적의 인덱스가 필요하며, 이를 위해 다음 능력이 필요하다.

- 인덱스의 물리적인 구조를 이해
- 복잡한 SQL을 분해해서 이해할 수 있는 능력
- 만들어진 인덱스가 어떻게 사용될지 예측 가능한 능력
- 테이블 내의 데이터 속성을 파악할 수 있는 능력
- JOIN의 내부적인 처리 방법(NESTED LOOPS, MERGE, HASH)의 이해

---

# 6.1 인덱스

인덱스 : '테이블 내의 ***데이터를 찾을 수 있게 일부 데이터를 모아서 구성한 데이터 구조***'

인덱스를 사용해서, 테이블 내의 데이터를 빠르게 찾아낼 수 있다. 

→  책에 붙인 포스트잇을 떠올려보자😀
<img width="543" height="547" alt="image" src="https://github.com/user-attachments/assets/65ba029a-aa99-474e-8e1d-258866cd6008" />
```sql
CREATE TABLE T_ORD_BIG AS
SELECT T1.*, T2.RNO, TO_CHAR(T1.ORD_DT,'YYYYMMDD') ORD_YMD
FROM T_ORD T1 ,(SELECT ROWNUM RNO FROM DUAL CONNECT BY ROWNUM<=1000)T2;

-- T_ORD_BIF 테이블의 통계를 생성하는 명령어-- 
-- 첫 번째 파라미터에는 테이블 OWNER 두번쨰 파라미터에는 테이블 명을 입력
EXEC DBMS_STATS.GATHER_TABLE_STATS('ORA_SQL_TEST','T_ORD_BIG');
```

- T_ORD_BIG 테이블에 3천만 건 정도의 데이터가 입력된다.
- RNO는 인덱스 테스트만을 위한 숫자값
- 다음으로 COUNT(*)에 대한 실행계획을 살펴보자
<img width="585" height="152" alt="image" src="https://github.com/user-attachments/assets/6a0187d5-5526-4104-b11e-740fd92183b0" />
TABLE ACCESS FULL 작업을 한 뒤에, SORT AGGREGATE처리를 하고 있다.

테이블 전체를 읽어서 ORD_SEQ가 343인 데이터를 찾아내서 카운트 처리를 하는 것이다.

이제 인덱스를 만들고 다시 해보자

```sql
CREATE INDEX X_T_ORD_BIF_TEXT ON T_ORD_BIG(ORD_SEQ);
```
<img width="595" height="164" alt="image" src="https://github.com/user-attachments/assets/941e67fd-8242-4845-95b1-0bd1cfd4b065" />
Buffers가 기존 258K에서 24로 좋아졌다.

또한 ACCESS FULL이 아닌, INDEX RANGE SCAN으로 바뀌었다. ORD_SEQ가 343인 데이터를 찾기 위해 인덱스를 이용한 것이다.

→ 인덱스를 사용함으로써 성능 개선된 것 확인. 

---

## 인덱스의 종류

### 1. 인덱스를 **구성하는 컬럼 수**에 따라

- **단일 인덱스** : 인덱스에 하나의 컬럼만 사용 (주로 고객 ID, 주문번호 같은 PK)
- **복합 인덱스** : 인덱스에 두 개 이상의 컬럼을 사용

📍잘 만들어진 복합 인덱스는 여러 인덱스를 대신할 수 있으며, 여러 SQL의 성능을 커버할 수 있다.

📍가능하면 하나의 복합 인덱스로 여러 SQL을 커버하는 것이 좋다.

### 2. 인덱스를 구성하는 **컬럼 값들의 중복 허용 여부**에 따라

- **유니크 인덱스** : 인덱스 구성 컬럼들 값에 중복 허용 x
- **비유니크 인덱스** : 인덱스 구성 컬럼 값에 중복을 허용

데이터 설계 시점부터 *업무적으로 유니크한 속성들을 파악해서 유니크 인덱스로 만들어 주는 것이 좋다.*

## 인덱스의 물리적인 구조

- **B*트리 인덱스**
- **비트맵 인덱스** : 값 종류가 많지 않을 경우 (주문 유형에 대한 값이 주문 대기 주문 완료 두 종류일 경우)
- **IOT** : 테이블 자체를 특정 컬럼을 기준으로 인덱스화 (MYSQL 클러스터드 인덱스) - B*트리구조로 만들어짐

📍대용량 테이블에서는 파티션을 구성하는 것이 좋다. 파티션 없이 인덱스를 만들어 사용하기엔 한계가 있다.

📍오래된 데이터는 별도 저장소로 백업한 후 주기적으로 지우는 것도 비용과 성능에 도움이 된다.

📍파티션 테이블에는 파티션 된 인덱스를 만들 수 있다

---

### B* 트리 구조와 탐색 방법

인덱스를 생성할 대 **별다른 옵션을 정의하지 않으면, B*트리 구조의 인덱스가 만들어진다.**

B*트리의 B는 균형이 잡혀있다는 뜻이다. (Balanced tree) 이는 리프 노드들이 같은 깊이에 자리해 있다는 뜻이다.

‘*’은 근접한 리프노드가 연결된 자료구조임을 뜻한다.

B*트리는 *균형이 잡혀 있고, 근접한 리프 노드가 연결된 구조*이다.
<img width="605" height="262" alt="image" src="https://github.com/user-attachments/assets/010c3bdc-b178-4d78-8415-d104ae3e1fd9" />
인덱스를 구성하는 블록은 인덱스 블록이라고한다. 이들은 서로 연결되어 있다.

- **루트 블록**

: 최상위 단 하나만 존재

: 하위 브랜치 블록의 인덱스 키 값과 주소를 가짐

- **브랜치 블록**

: 루트와 리프의 중간 여러 층이 있을 수 있다.

: 하위 브랜치의 인덱스 키 값과 주소 또는 하위 리프의 키 값과 주소를 가짐

- **리프 블록**

: 최하위에만 위치

: 인덱스 키 값과 데이터 로우 위치를 가짐

: 리프 블록은 인덱스 키 값 순으로 정렬되어 있음
<img width="1462" height="822" alt="image" src="https://github.com/user-attachments/assets/bf8feeb3-1c7f-435e-9671-8e7552bb5156" />
*ORD_TMD로 구성된 인덱스를 통해 ORD_YMD = 20170104를 찾는 과정*

1. **루트 블록**
<img width="424" height="178" alt="image" src="https://github.com/user-attachments/assets/ecb0da1f-eb97-4c0a-8623-2e84386e98fe" />
ORD_YMD로 구성된 인덱스의 **루트 블록**이다. 세 개의 브랜치 블록(B05,B06,B01)을 찾아갈 수 있다.

찾으려는 20170104는 빈값보다 크고 20180601보다는 작다. 그러므로, 빈 값의 브랜치 블록으로 이동한다.

2. **브랜치 블록**
<img width="363" height="157" alt="image" src="https://github.com/user-attachments/assets/eb1fab8a-93a3-409a-af4d-161805fa8b98" />
B05 블록은 하위에 세 개의 리프 블록을 가지고 있다. 찾으려는 값은 B10보다 크고, B21보다 작거나 같다 따라서 B10으로 이동해야한다.

3. **리프 블록 스캔**

B10 블록의 마지막 부분에 20170104가 있다.

<img width="535" height="302" alt="image" src="https://github.com/user-attachments/assets/6f2d0fc2-ef76-4a0d-bebd-4a22ef823958" />

인덱스를 검색해서 리프 블록에 도달하면, 리프 블록을 차례대로 스캔한다.

스캔은 찾으려는 값보다 큰 값을 발견하기 전까지 수행한다. (20170105만날 때 그만둠)

이때 리프 블록을 스캔하면서 ROWID를 참고해 실제 테이블에 접근하는 작업을 수행한다.

(ROWID는 데이터가 실제 저장된 주소 값이다) ROWID를 이용해 데이터를 찾아내는 과정은 테이블 실행계획에

TABLE ACCESS BY INDEXT ROWID 오퍼레이션으로 나타난다.

---

### 데이터를 찾는 방법

- **테이블 전체 읽기 (TABLE ACCESS FULL):**
    - 테이블 전체 읽기는 테이블의 **데이터 블록을 차례대로 모두 읽으면서 필요한 데이터를 찾는 방법**
- **인덱스를 이용한 찾기(INDEX RANGE SCAN & TABLE ACCESS BY INDEX ROWID)**
    - 인덱스를 이용해 필요한 데이터만 찾기
- **ROWID를 이용한 직접 찾기(TABLE ACCESS BY INDEX ROWID)**
    - 테이블의 레**코드 주소인 ROWID를 조건 값으로 직접 찾아가는 방식**이다.
    

📍테이블 전체 읽기 : 실행계획에 TABLE ACCESS FULL로 표시됨

- 찾고자 하는 조건에 활용할 인덱스가 없거나
- 인덱스보다 테이블 전체를 읽는 것이 효율적이라고 판단할 대 사용하는 방법이다.

오라클에서 데이터가 테이블에 저장될 때는 특정 순서를 갖지 않는다.

<img width="299" height="364" alt="image" src="https://github.com/user-attachments/assets/6cac157a-1d03-4fbf-9939-ddfd26edf5e4" />
이와 같은 상황에서 오라클에서 데이터를 찾는다면, 어디 있는지 정확하게 알 수 없음으로 모두 읽는다.

WHERE 조건절에 사용된 컬럼에 대해 적절한 인덱스가 없다면 테이블 전체 읽기가 발생한다.

**만약 천만 건에서 데이터를 찾는다면, 모두 읽어야하므로 비효율이 발생할 수도 있다.**

다만 **천만 건의 데이터를 위한 백만건의 인덱스가 있다면, 차라리 TABLE ACCESS FULL이 더 효율적일 수도 있다.**

무조건 성능이 나쁘다고 오해는 말자 !

## **인덱스를 활용한 찾기**

인덱스로 데이터를 찾는 방법은 INDEX RANGE SCAN, INDEX SKIP SCAN, INDEX FULL SCAN등 다양한 방법이 있다.

가장 기본은 INDEX RANGE SCAN이다.

<img width="519" height="241" alt="image" src="https://github.com/user-attachments/assets/7a2ae145-852f-4a8d-a448-a9982a76592b" />
- 루트에서 리프로: 검색 조건에 해당하는 첫 리프 블록을 찾는 과정
- 리프 블록 스캔 : 찾아낸 지점부터 리프 블록을 차례대로 읽어 가는 과정
- 테이블 접근 : 리프 블록을 스캔하면서 필요 따라 테이블에 접근하는 과정

**1) 루트에서 리프로**

루트 블록에서 주어진 조건이 저장된 리프 블록을 찾아가는 과정이다.

이 과정은 부하가 없다고 생각될 정도로 매우 빠르게 이루어진다.

**2) 리프 블록 스캔(RANGE SCAN)**

리프 블록은 인덱스 키 컬럼의 값으로 정렬되어 있다. 루트에서 리프 과정에서 3번 리프 블록을 찾았다하면, 이후로 데이터를 차례대로 읽어 나간다. 차례대로 읽는 과정은 ORD_YMD가 조건보다 큰 값을 만날 때 까지이다.
<img width="602" height="165" alt="image" src="https://github.com/user-attachments/assets/cfe2fd8a-d36d-4018-9719-13996343f44d" />
**3) 테이블 접근**

리프 블록 스캔 과정에서 필요에 따라 테이블 접근을 한다. 인덱스 리프 블록의 ROWID 값을 참조해 테이블의 데이터를 찾아가는 과정이다.

이 과정은 테이블에서 필요한 값이 있을 때만 일어난다.

만약 ORD_YMD 값만 사용해 SQL을 처리할 수 있다면, 이 과정은 생략한다.
<img width="593" height="224" alt="image" src="https://github.com/user-attachments/assets/0e4a9929-042a-47f9-a698-5f4056117a75" />
```
CREATE INDEX X_T_ORD_BIG_1 ON T_ORD_BIG(ORD_YMD);

SELECT/**GATHER_PLAN_STATISTICS INDEX(T1 X_T_ORD_BIG_1)*/
T1.CUS_ID, COUNT(*) ORD_CNT
FROM T_ORD_BIG T1
WHERE T1.ORD_YMD = '20170316'
GROUP BY T1.CUS_ID
ORDER BY T1.CUS_ID;
```
<img width="601" height="129" alt="image" src="https://github.com/user-attachments/assets/74ed8cd1-fe77-43d0-9b69-78ee64bbb8ff" />
위 SQL을 실행하고, 실행계획을 살펴보면  INDEX RANGE SCAN과 TABLE ACCESS BY INDEX ROWID 작업이 수행된다.

WHERE 조건에 맞는 데이터를 찾기 위해 INDEX RANGE SCAN이 사용되었고, 인덱스에 없는 CUS_ID값을 가져오기 위해 TABLE ACCESS BY INDEX ROWID 작업이 수행된 것이다.
<img width="454" height="317" alt="image" src="https://github.com/user-attachments/assets/23963325-15dd-4042-97f9-65883131d4fd" />
---

## INDEX RANGE SCAN VS TABLE ACCESS FULL

데이터를 찾을 때, 인덱스를 이용한 찾기와 테이블 전체 읽기의 성능을 비교해보자

- 비교에 앞서, 랜덤 액세스 - IO 작업 한 번에 하나의 블록을 가져오는 접근 방법을 의미, 인덱스의 리프 블록에서 ROWID를 이용해 테이블에 접근할 대 랜덤 액세스가 발생한다.

찾으려는 데이터가 많지 않으면, 랜덤 엑세스가 나쁜 방법은 아니지만, 데이터가 많아지면 엑세스는 오히려 비효율적이다. 

```sql
SELECT/**GATHER_PLAN_STATISTICS*/
T1.CUS_ID, COUNT(*) ORD_CNT
FROM T_ORD_BIG T1
WHERE T1.ORD_YMD = '20170316'
GROUP BY T1.CUS_ID
ORDER BY T1.CUS_ID;
```

위 커리는 ORD_YMD 인덱스를 이용했을 때 성능이 더 좋은 SQL이다.

주문 데이터는 총 3천만 건 정도의 데이터가 있다. 이 중 ORD_YMD가 20170316인 데이터는 5만건이다. 3천만건에서 5만건에서 5만 건 정도를 찾는 경우라면, INDEX_RANGE SCAN이 효율적이라고 판단할 수 있다.

- 여기서 5만건 이라는 수치는 모든 상황에서 절대적인 수치가 아니다. 서버와 스토리지 성능 테이블 구조 SQL에 따라 달라진다.

이번에는 T_ROD_BIG 테이블에서 3개월간의 주문을 조회해보자 (약 7,650,000건)
```sql
SELECT/**GATHER_PLAN_STATISTICS INDEX(T1 X_T_ORD_BIG_1)*/
T1.ORD_ST, SUM(T1.ORD_AMT)
FROM T_ORD_BIG T1
WHERE T1.ORD_YMD BETWEEN '20170401' AND '20170630'
GROUP BY T1.ORD_ST;
```

<img width="594" height="143" alt="image" src="https://github.com/user-attachments/assets/2c53ca51-fbc3-460f-9298-6d9cf4c23b1d" />


위와 같은 실행계획이 나온다. 이때 TABLE ACCESS BY INDEX ROWID가 7,650K번 실행되었다.

이는 바로 전 단계 (INDEX RANGE SCAN)의 A-Rows만큼 실행된 것이다.

매우매우 많은 랜덤 엑세스가 발생했다.
이제 SQL을 FULL 힌트로 사용해보자
```sql
SELECT/**GATHER_PLAN_STATISTICS FULL(T1)*/
T1.ORD_ST, SUM(T1.ORD_AMT)
FROM T_ORD_BIG T1
WHERE T1.ORD_YMD BETWEEN '20170401' AND '20170630'
GROUP BY T1.ORD_ST;
```

<img width="581" height="128" alt="image" src="https://github.com/user-attachments/assets/2cffd2ed-692c-42b7-964b-bb39b98c99e8" />


총 시간이 4.77로 단축되었고, Bufferes도 253K로 좋아졌다.

**찾고자 하는 데이터가 특정 수준 이상으로 많으면 인덱스를 이용한 랜덤 엑세스보다 FULL SCAN방식이 훨씬 효율적이다.**

- 데이터가 계속 쌓이는 구조라면 FULL SCAN방식이 시간이 지날 수록 성능이 떨어진다.
    - 따라서 오래된 데이터를 잘라내거나 파티션 전략을 수림할 필요가 있다.
- 중간 집계 테이블을 활용하는 방안을 고려해야한다.

---

- 적은 양의 데이터를 읽는다면 INDEX RANGE SCAN이 유리
- 많은 양의 데이터를 읽는다면 FULL SCAN이 유리
- FULL SCAN은 데이터가 쌓일수록 성능이 나빠지므로, 추가적인 테이블 관리 전략이 필요하다.
# 6.2 단일 인덱스

### **단일 인덱스의 컬럼 정하기**

인덱스는 조건에 맞는 데이터를 빠르게 찾기 위한 객체이다.

**WHERE 조건절에 사용된 컬럼에 인덱스를 구성하는 것이 기본** 원리다.

```sql
SELECT/*+ GATHER_PLAN_STATISTICS*/
TO_CHAR(T1.ORD_DT,'YYYYMM'), COUNT(*)
FROM T_ORD_BIG T1
WHERE T1.CUS_ID = 'CUS_0064'
AND T1.PAY_TP = 'BANK'
AND T1.RNO = 2
GROUP BY TO_CHAR(T1.ORD_DT,'YYYYMM');
```

이제 **성능 개선을 위해 단 하나의 단일 인덱스를 고려**해보자

후보 컬럼은 WHERE 조건절에 사용된 CUS_ID, PAY_TP, RNO컬럼이다.

```sql
SELECT 'CUS_ID' COL, COUNT(*) CNT FROM T_ORD_BIG T1 WHERE T1.CUS_ID = 'CUS_0064'
UNION ALL
SELECT 'PAY_TP' COL, COUNT(*) CNT FROM T_ORD_BIG T1 WHERE T1.PAY_TP = 'BANK'
UNION ALL
SELECT 'RNO' COL, COUNT(*) CNT FROM T_ORD_BIG T1 WHERE T1.RNO = 2
```

<img width="205" height="150" alt="image" src="https://github.com/user-attachments/assets/e514ad71-ae66-4b2a-a7a1-519c141f2b19" />


**인덱스 컬럼 선정의 규칙 중 하나는 선택성이 좋은 컬럼을 사용하는 것**

주어진 ***조건에 데이터가 적을수록 선택성이 좋고, 조건에 해당하는 데이터가 많을수록 선택성이 나쁘다.***

RNO = 2 조건의 결과가 3047건으로 가장 적다. RNO 조건이 선택성이 가장 좋고, 그 다음 CUS_ID가 좋다.

성능을 위한 인덱스를 하나 만들어야 한다면, RNO 컬럼에 만드는 것이 좋다.

```sql
CREATE INDEX X_T_ORD_BIG_2 ON T_ORD_BIG(RNO);

SELECT/*+ GATHER_PLAN_STATISTICS INDEX (T1 X_T_ROD_BIG_2) */FROM T_ORD_BIG T1
WHERE T1.CUS_ID = 'CUS_0064'
AND T1.PAY_TP ='BANK'
AND T1.RNO =2
GROUP BY TO_CHAR(T1.ORD_DT ,'YYYYMM');
```
<img width="798" height="207" alt="image" src="https://github.com/user-attachments/assets/fa615ade-2b7c-4190-af5e-a6db6a77c8c8" />
위 SQL을 결과에 대한 실행 계획으로 위에 테이블을 얻을 수 있다.

- X_T_ORD_BIG_2 인덱스를 이용해 조건에 맞는 데이터를 찾는다. -> 3047건
- 찾아낸 3047건에 대해 TABLE ACCESS BY INDEX ROWID를 수행 -> 2건의 결과
- 마지막으로 GROUP BY를 수행

```sql
CREATE INDEX X_T_ORD_BIG_3 ON T_ORD_BIG(CUS_ID);
```

인덱스를 변경하고 같은 SQL을 돌려보자 결과는 아래와 같다.

<img width="794" height="188" alt="image" src="https://github.com/user-attachments/assets/22fb570d-ca70-43eb-94ea-677fda5e61f9" />


이전보다 오래 걸링 원인은 INDEX RANGE SCAN 단계의 A-Rows가 340K이므로, TABLE ACCESS BY INDEX ROWID가 340k번 수행된다.

<img width="772" height="313" alt="image" src="https://github.com/user-attachments/assets/fb267dcd-c509-4729-991b-cedac26d6972" />


RNO가 2인 데이터는 3047건이므로 TABLE ACCESS BY INDEX ROWID도 3047번이다. 하지만, CUS_ID는 340000건이므로 테이블 엑세스도 340000번 수행된다.

→ 선택성 좋은 RNO 칼럼에 인덱스를 생성했을 때 가장 좋은 효과 💡

---

### **단일 인덱스 VS 복합 인덱스**

**복합 인덱스는 단일 인덱스를 능가하는 성능을 낼 수 있다.** 하나의 복합 인덱스로 여러 개의 인덱스를 대신 할 수 있는 장점도 있다.

**인덱스가 많아질수록 입력, 수정 삭제 성능 감소가 발생**한다.

이와 같은 이유로 복합 인덱스를 이용해 인덱스의 수를 줄이는 것이 매우 중요하다.

```sql
DROP INDEX X_T_ORD_BIG_3;--인덱스 제거SELECT/*+ GATHER_PLAN_STATISTICS INDEX(T1 X_T_ORD_BIG_1) */
T1.ORD_ST, COUNT(*)
FROM T_ORD_BIG T1
WHERE T1.ORD_YMD LIKE '201703%'
AND T1.CUS_ID = 'CUS_0075'
GROUP BY T1.ORD_ST;
```
<img width="807" height="210" alt="image" src="https://github.com/user-attachments/assets/7786b826-c607-44f2-8b4a-8cf72f825f62" />
실행 계획을 보면,  실행시간은 6.61초, Bufferes는 338K, INDEX RANGE SCAN은 1850K이다

이번에는 WHERE 조건절에 사용된 ORD_YMD와 CUS_ID 컬럼 두 개를 모두 포함하는 복합 인덱스를 만들어보자

```sql
CREATE INDEX X_T_ORD_BIG_3 ON T_ORD_BIG(ORD_YMD,CUS_ID);
```

```sql
SELECT/*+ GATHER_PLAN_STATISTICS INDEX(T1 X_T_ORD_BIG_3) */
T1.ORD_ST, COUNT(*)
FROM T_ORD_BIG T1
WHERE T1.ORD_YMD LIKE '201703%'
AND T1.CUS_ID = 'CUS_0075'
GROUP BY T1.ORD_ST;
```

<img width="793" height="192" alt="image" src="https://github.com/user-attachments/assets/ae918a58-5ca3-4110-95fd-6a90a918434d" />


실행계획을 확인하면, 전체 실행 시간이 1.49초로 개선되었다.

INDEX RANGE SCAN의 A-Rows가 30000에 불과하다.
<img width="758" height="544" alt="image" src="https://github.com/user-attachments/assets/49a8353b-b74d-4c20-bc68-14dd2fff3ebd" />
복합 인덱스를 사용한 리프 블록을 보면, ORDER BY 결과처럼 ORD_YMD별로 정렬된 후 CUS_ID로 데이터가 정력되어 있다.

단일 인덱스의 경우 ORD_YMD 조건은 인덱스를 이용했지만, CUS_ID를 확인하기 위해서는 테이블에 방문해야만 알 수 있다.

→ 복합 인덱스 사용 시 성능이 훨씬 좋은 것 확인 💡
