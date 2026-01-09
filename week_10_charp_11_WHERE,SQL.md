이번장 핵심 키워드
데이터베이스가 최소한의 일을 하도록 돕자

# 11.1 WHERE 절 가이드

## **컬럼 변형 금지 = 인덱스 망가트리지 마라**

SUBSTR, LOWER 등으로 컬럼을 변형하면 인덱스 사용 불가

**❌ 나쁜 예**

```sql

-- 컬럼을 가공하면 인덱스를 못 씀
WHERE UPPER(name) = 'JOHN'
WHERE SUBSTRING(order_date, 1, 4) = '2024'
WHERE price * 1.1 > 100

```

**✅ 좋은 예**

```sql

-- 컬럼은 그대로 두고 조건값을 바꿈
WHERE name = 'JOHN'-- 대소문자 구분이 문제면 데이터 정리나 다른 방법 고려
WHERE order_date LIKE '2024%'
WHERE price > 100/1.1
```

## **자료형을 맞춰라!**

컬럼과 같은 자료형의 조건값 사용으로 자동 형변환 방지

**❌ 자료형 불일치**

```sql

-- customer_id가 문자열인데 숫자로 비교
WHERE customer_id = 123

-- order_date가 문자열인데 날짜로 비교
WHERE order_date = DATE('2024-01-01')

```

**✅ 자료형 일치**

```sql

WHERE customer_id = '123'
WHERE order_date = '2024-01-01'

```

데이터베이스가 자동 변환하느라 고생하지 않도록 처음부터 맞는 타입을 쓰자…

자료형이 안 맞으면  ⇒ 인덱스도 쓰지 못함 ⇒ 테이블 풀스캔 발생!! ⇒ 효율이 급격히 떨어진다!

### 특히 날짜에 주의할것

```sql
// 날짜가 문자형으로 저장된 경우 (YYYYMMDD, YYYY-MM-DD)
order_date VARCHAR(10) = '2024-03-15'

// 날짜가 날짜형으로 저장된 경우
order_date DATE = '2024-03-15'
order_date DATETIME = '2024-03-15 14:30:00'
```

```sql
// 문자형 컬럼에 날짜형 조건값 => die💥
WHERE order_date = DATE('2024-03-15') ⇒  컬럼 자동 변환…

// 날짜형 컬럼을 문자로 변환 => die💥
WHERE DATE_FORMAT(order_date, '%Y-%m-%d') = '2024-03-15'

// 시분초가 있는 날짜에 잘못된 조건 => die💥
WHERE order_datetime = '2024-03-15' ⇒ 00:00:00만 매칭됨
```

### 실무 팁~

- 날짜 범위 조건에서는 `>=`와 `<` 조합이 안전
- `BETWEEN`은 양쪽 끝을 포함하므로 시간 부분에서 예상치 못한 결과 가능
- 시스템 전체적으로 날짜 저장 형식을 통일하는 것이 중요
- 개발 초기에 테이블 설계할 때 자료형을 신중히 선택
- 애플리케이션에서 파라미터 타입을 테이블 컬럼과 일치시키기
- 숫자처럼 보이는 코드값(우편번호, 계좌번호 등)도 문자형이 적절할 수 있음

## **긍정적으로 사고하기~**

(럭키비키..?)

부정형 조건(`NOT`, `!=`, `NOT IN`)은 데이터베이스가 이것 말고 나머지 전부를 찾아야 해서 비효율적

긍정형 조건은 정확히 이것들을 찾으므로 인덱스 활용도가 높아진다~

**❌ 부정형 조건**

```sql
WHERE status NOT IN ('cancelled', 'pending', 'failed')
WHERE type != 'premium'
```

**✅ 긍정형 조건**

```sql
WHERE status IN ('completed', 'shipped')
WHERE type = 'basic'
```

NOT IN은 조건의 풀스캔 ⇒ IN은 일치하는게 나오면 그만 읽어도 판별이 되지만 NOT IN 은 조건을 끝까지 읽어야 함

---

🧐

나의 생각

그렇다면 IN 안에 들어가는 순서도 선택도가 높은 순으로 하는게 더 성능에 유리할까?

RDBMS는 IN 조건을 A = x OR A = y OR A = z ...처럼 내부적으로 처리하는 경우가 있고,

이 경우는 **왼쪽부터 차례로 비교하면서 일치 여부를 확인하므로 성능에 영향이 있다!**

| 항목 | **Oracle** | **PostgreSQL** | **MySQL** |
| --- | --- | --- | --- |
| **내부 처리 방식** | - `IN` 리스트를 **B-tree 인덱스**와 함께 효율적으로 탐색함- 내부적으로 `= OR = OR =`로 전개하기도 함 | - `IN` 리스트를 **HashSet** 형태로 최적화- 값이 많으면 **배열 스캔** | - 단순한 경우 `= OR = OR =`로 **순차적 비교**- 최적화가 부족해 순서 영향 **클 수 있음** |
| **순서의 영향** | ❌ 거의 없음(오라클 옵티마이저가 자동 최적화함) | ❌ 거의 없음(`IN`을 내부 배열로 정렬하거나 Hash 구조로 처리) | ✅ **있을 수 있음**특히 `OR`로 전개되어서 앞에 선택도 높은 값이 빠르면 조기 종료됨 |
| **대량 값 처리 (100+개)** | - 옵티마이저가 내부적으로 **임시 테이블 처리**- 경우에 따라 성능 저하 | - 너무 크면 **배열 검색 비용 커짐**- `JOIN` 방식 권장 | - `IN` 항목 수 많으면 **WHERE 조건 복잡도 증가**- `JOIN` 또는 `EXISTS`로 리팩터링 권장 |
| **인덱스 사용 여부** | ✅ 가능단, 인덱스 컬럼에 `IN` 적용시 | ✅ 가능조건에 따라 `Index Scan` 또는 `Bitmap Index Scan` | ✅ 가능하지만 `IN` 값이 많으면 풀스캔 유발 가능 |
| **실행계획에서 표현** | `IN-list iterator`, `FILTER`, `INDEX RANGE SCAN` 등 | `Seq Scan`, `Bitmap Index Scan`, `Index Only Scan` 등 | `range`, `ref_or_null`, `index_merge`, `ALL` 등 |
| **최적화 팁** | - `IN`보다 `JOIN`/`SEMI JOIN`이 나은 경우도 있음 | - 작은 리스트는 OK큰 리스트는 `JOIN` / 서브쿼리로 | - 리스트가 짧을수록 유리순서 주의, 필요 시 `EXISTS`나 `JOIN`으로 전환 |

그러나 거의 영향이 없는 것 같다…

---

- `WHERE status_code IN (...)`은 **B-tree 인덱스 활용 가능**
- `NOT IN (...)`은 PostgreSQL에서는 **Index Scan 사용이 어려운 경우 많음**

> NOT EXISTS나 LEFT JOIN + IS NULL로 바꾸는 게 성능 개선에 도움이 된다고 함
> 

### `NOT EXISTS`는 **인덱스를 타고 조기 종료** 가능

```sql
SELECT * FROM A
WHERE NOT EXISTS (
  SELECT 1 FROM B WHERE B.key = A.key
);
```

- `B.key`에 인덱스가 있다면 → **빠르게 탐색 가능**
- A의 각 row마다 **B에서 해당 키만 조회하면 되므로**, 매우 효율적
- **해당 키가 존재하면 바로 제외**되므로 **조기 종료**가능

### ❌ 2. `NOT IN`은 **전체 리스트 비교 + NULL 리스크**

```sql
SELECT * FROM A
WHERE key NOT IN (SELECT key FROM B);
```

- B의 결과를 **전체 비교**해야 함 → **조기 종료 불가능**
- B에 **NULL이 1개라도 있으면 → 전체 결과는 무조건 FALSE or UNKNOWN**
- 일부 DB는 이걸 우회하려고 **임시 테이블 스캔 or Full Scan**을 하기도 함

---

### `LEFT JOIN + IS NULL`도 빠른 편

```sql
SELECT A.*
FROM A
LEFT JOIN B ON A.key = B.key
WHERE B.key IS NULL;
```

- `B.key`에 인덱스가 있으면 → **Index Lookup으로 빠르게 JOIN**
- B에 일치하는 값이 없다는 걸 확인만 하면 되므로, **불필요한 스캔 없이 최적화 가능**
- 일부 DB에서는 `NOT EXISTS`와 동일한 `ANTI JOIN` 전략을 씀(Postgre)

### 실무 팁~

- **상태 코드 설계 시 고려!** 자주 조회되는 조건이 긍정형이 되도록 설계
- **비즈니스 로직 파악!** "활성 사용자", "유효한 주문" 등 긍정적 조건으로 사고
- **코드 테이블 활용!** 상태값들을 코드 테이블로 관리해서 조건 구성하기

---

🧐

코드 테이블이란?

상태 코드(status code)를 숫자나 문자열로 관리하는 참조 테이블(reference table)

| status_code | status_name | active_flag |
| --- | --- | --- |
| 100 | 주문완료 | Y |
| 200 | 배송중 | Y |
| 300 | 배송완료 | Y |
| 400 | 주문취소 | N |
| 900 | 시스템 오류 | N |

코드 테이블은 오류가 아닌 조건을 긍정문으로 쓸 수 있게 만들어준다!!

혁신적…

❌ 매번 부정 조건 쓰면 복잡하고 비효율
WHERE status_code NOT IN (400, 900)

 ✅ 긍정 조건으로 바꿔서 명확하고 인덱스 효율도 좋음
WHERE status_code IN (100, 200, 300)

비즈니스 의미를 쿼리에서 분리할 수 있는 면에서도 좋음!

❌ 하드코딩된 조건은 변경 시 코드 전체 수정 필요
WHERE status_code IN (100, 200, 300)

✅ 코드 테이블에서 의미로 분리
WHERE osc.active_flag = 'Y'

**코드 수정은 테이블만 바꾸면 끝**

쿼리는 "유효 상태"라는 개념만 다룸

---

## **LIKE 최소화**: 불필요한 LIKE 대신 = 조건 사용

`LIKE`는 패턴 매칭이라 `=`보다 처리 비용이 높다

```sql
-- 선택적 조건 처리를 위한 LIKE 남용
WHERE customer_id LIKE @customer_id + '%'    -- @customer_id가 ''이면 모든 고객
WHERE order_status LIKE @status + '%'       -- @status가 ''이면 모든 상태
WHERE product_name LIKE '%' + @keyword + '%' -- @keyword가 ''이면 모든 상품

-- 정확한 값 비교인데 LIKE 사용
WHERE order_id LIKE 'ORD123456'  -- = 'ORD123456'와 동일한 결과인데 느림
WHERE status LIKE 'ACTIVE'       -- = 'ACTIVE'가 더 빠름
```

**✅ 적절한 조건 사용**

```sql
-- 동적 쿼리로 조건 분기 (애플리케이션 레벨)
IF @customer_id IS NOT NULL
    WHERE customer_id = @customer_id
-- ELSE 조건 없음 (전체 조회)-- 정확한 매칭은 등호 사용
WHERE order_id = 'ORD123456'
WHERE status = 'ACTIVE'

-- 진짜 패턴 매칭이 필요한 경우만 LIKE
WHERE product_name LIKE '%phone%'-- 상품명에 'phone' 포함
WHERE email LIKE '%@company.com'-- 회사 이메일 도메인
WHERE phone LIKE '010-%'-- 010으로 시작하는 번호

-- 전체 조회가 필요하면 조건 자체를 제거
-- WHERE 1=1 같은 무의미한 조건 지양

```

### 실무 팁~

- 애플리케이션에서 조건 유무에 따라 WHERE절 자체를 다르게 구성
- MyBatis, JPA Criteria 등으로 조건부 쿼리 생성
- **복합 인덱스 고려 ⇒** LIKE를 꼭 써야 한다면 전방 일치(prefix) 패턴으로 인덱스 활용

---

🧐

prefix 패턴 ⇒ 선택도 높음 초반이 아닌 것 빠르게 제거

suffix 패턴 ⇒ 선택도 낮음, 풀스캔 발생 초반이 맞지 않아도 뒤까지 봐야 함

내 생각 : suffix는 뒤부터 읽으면 prefix랑 동일하게 동작하는거 아냐?!

왜 역으로는 안 읽지?!

왜 LIKE '%abc'는 뒤에서부터 읽어서 인덱스를 활용하지 못할까?

PostgreSQL/MySQL/Oracle 모두 기본적으로 **B-tree 인덱스**

B-tree는 **왼쪽 → 오른쪽** 정렬 구

### DB 내부

사전순으로 정렬되어있음

aaron
abc
abcde
abcdx
baker
barry

### 동작 방식

시작 위치 'abc' 찾고 → 그 이후로 abc*, abcd*, abce* 등 순서대로 읽기

= 조건(equal) 후 like 조건(range)로 동작~

**B-tree 인덱스는 뒷부분으로 정렬되어 있지 않음**.

즉, `'barabc'`, `'helloabc'`, `'zzzabc'`는 인덱스 구조상 **전혀 모여 있지 않고 흩어져 있어서 한번에 보기 어려움 ⇒ 테이블 풀스캔 발동~**

그러니까 왜 인덱스 생성 옵션중에 뒤부터 읽는 인덱스 만드는 조건이 없냐고~

내가 뒤집어서 저장하고 저 칼럼에 인덱싱을 걸면 suffix를 prefix처럼은 쓸 수 있겠다… 

그렇게 두번 저장하나 같은 칼럼 인덱스 두배로 만드나 비슷하니까 없는걸까?

⇒ 인덱스 만드는 방법이 존재했다~

### **역방향 인덱스 (reverse index)** 또는 **함수 기반 인덱스**!

### PostgreSQL 예시

```sql

-- 문자열을 거꾸로 뒤집어서 인덱스 만들기
CREATE INDEX idx_reverse_name ON users (reverse(name));

-- 검색 시에도 문자열을 reverse해서 검색
SELECT * FROM users WHERE reverse(name) LIKE 'cba%';

```

---

# 11.2 불필요한 부분 제거

## 존재 여부만 확인할 때는 한 건만 보기

**❌ 비효율적**

```sql

-- 1000건이 있어도 모두 세어버림
SELECT COUNT(*) FROM orders WHERE customer_id = 1
```

**✅ 효율적**

```sql

-- 한 건만 확인하고 끝
SELECT 1 FROM orders WHERE customer_id = 123 LIMIT 1

-- 또는 EXISTS 사용
SELECT CASE WHEN EXISTS(
  SELECT 1 FROM orders WHERE customer_id = 123
) THEN 1 ELSE 0 END

```

있는지 없는지만 알면 되는데 모든걸 세는 일은 하지말자 

---

🧐

나의 생각 ⇒ JPA에서 exists쓰지 않고 findById()로 엔티티 리턴하는것도 마찬가지?

✅

**JPA에서 `findById()`를 사용하는 건 `SELECT *`로 엔티티 전체를 불러오는 것**

이라서, 존재 여부만 알고 싶을 때는 **비효율적임

Optional<User> user = userRepository.findById(123L);**

얘는 내부적으로

SELECT * FROM user WHERE id = 123;

이렇게 동작한다고 함…

**전체 컬럼을 모두 읽어서** 매핑까지 하고 엔티티 생성 ㅜㅜ

boolean exists = userRepository.existsById(123L);

이렇게 하면

SELECT 1 FROM user WHERE id = 123; 

이렇게 동작…

---

## **조인 최소화**: COUNT에서 불필요한 조인/ORDER BY 제거

COUNT는 "개수"만 필요한데, 화면 표시용 조인이나 정렬까지 포함하면 불필요한 작업이 발생

**❌ COUNT에 포함된 불필요한 작업들**
```sql

-- 페이징용 COUNT 쿼리 - 화면 표시용 조인까지 포함
SELECT COUNT(*) FROM (
    SELECT o.order_id, o.order_date,
           c.customer_name,-- 😱 COUNT에 고객명 불필요
           p.product_name,-- 😱 COUNT에 상품명 불필요
           s.supplier_name,-- 😱 COUNT에 공급업체명 불필요
           r.region_name-- 😱 COUNT에 지역명 불필요
    FROM orders o
    JOIN customers c ON o.customer_id = c.id-- 😱 불필요 조인
    JOIN products p ON o.product_id = p.id-- 😱 불필요 조인
    JOIN suppliers s ON p.supplier_id = s.id-- 😱 불필요 조인
    JOIN regions r ON c.region_id = r.id-- 😱 불필요 조인
    WHERE o.order_date BETWEEN '2024-03-01' AND '2024-03-31'
    ORDER BY o.order_date DESC, c.customer_name-- 😱 COUNT에 정렬 불필요
) t;

-- 실제 데이터 조회 쿼리는 위와 동일한 구조로 별도 실행

```

**✅ COUNT 최적화 - 필요한 것만**

```sql

-- COUNT는 최소한으로 - 조인 없이
SELECT COUNT(*)
FROM orders o
WHERE o.order_date BETWEEN '2024-03-01' AND '2024-03-31';

-- 조인이 결과 건수에 영향을 주는 경우만 포함-- 예: INNER JOIN으로 데이터가 필터링되는 경우
SELECT COUNT(*)
FROM orders o
JOIN customers c ON o.customer_id = c.id-- 활성 고객만 카운트하는 경우
WHERE o.order_date BETWEEN '2024-03-01' AND '2024-03-31'
  AND c.status = 'ACTIVE';-- 이 조건이 건수에 영향-- 실제 데이터는 전체 조인으로 별도 조회
SELECT o.order_id, o.order_date,
       c.customer_name, p.product_name,
       s.supplier_name, r.region_name
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
JOIN suppliers s ON p.supplier_id = s.id
JOIN regions r ON c.region_id = r.id
WHERE o.order_date BETWEEN '2024-03-01' AND '2024-03-31'
ORDER BY o.order_date DESC, c.customer_name
LIMIT 20 OFFSET 0;-- 페이징

```

### 실무 팁~
### 

- COUNT는 가볍게 써야 한다
    - `SELECT COUNT(*)`만 남겨라
    - `JOIN`, `ORDER BY`, `SELECT 컬럼` 다 제거해라
- 조인이 "필터"일 때만 예외

```sql

-- 예: 고객이 ACTIVE한 주문만 세는 경우
SELECT COUNT(*)
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.order_date BETWEEN '...' AND '...'
  AND c.status = 'ACTIVE';

// 이건 조인이 **결과 수를 바꾸므로 유지
//** 이럴 땐 PostgreSQL 옵티마이저가 Index Nested Loop Join 등을 써서 비교적 최적화
```

- 조회 쿼리는 별도로 작성 (페이징 포함)

```sql

SELECT o.order_id, c.customer_name, ...
FROM orders o
JOIN ...
ORDER BY ...
LIMIT 20 OFFSET 0;
// 이건 **사용자 화면 표시용
//COUNT는 이거랑 분리**해서 "전체 개수 계산"만 따로 처리

```

- 전체 카운트가 불필요할 땐 "더보기" 전략
    - 게시판에서 "총 몇 건"이 중요한 게 아니라면
    - `LIMIT 21`을 걸어서 "다음 페이지 있나?"만 판단

```sql
SELECT ... FROM orders
WHERE ...
ORDER BY ...
LIMIT 21;
-- 결과가 21개면 → 다음 페이지 있음
```

## 필요한 것만 가져와라
SELECT * 대신 명시적 컬럼 지정

**❌ 과도한 조회**

```sql

-- 모든 컬럼을 가져옴
SELECT * FROM products

-- 관련 없는 테이블까지 조인해서 COUNT
SELECT COUNT(*) FROM (
  SELECT p.*, c.name, s.address
  FROM products p
  JOIN categories c ON p.category_id = c.id
  JOIN suppliers s ON p.supplier_id = s.id
  WHERE p.price > 100
)

```

**✅ 필요한 것만**

```sql

-- 필요한 컬럼만
SELECT id, name, price FROM products

-- COUNT에는 조인 불필요
SELECT COUNT(*) FROM products WHERE price > 100

```

더 많이 가져올수록 더 느리다…

## 반복 작업을 줄이자

**서브쿼리를 통합해서** 동일 테이블 반복 접근 최소화

**❌ 같은 테이블을 여러 번 접근**

```sql

  customer_id,//빠삐코도 아니고 customers를 이렇게 많이 볼 필요 없잖아...?!
  (SELECT name FROM customers WHERE id = o.customer_id) as name, //=>한번보고
  (SELECT email FROM customers WHERE id = o.customer_id) as email,//=>두번보고
  (SELECT phone FROM customers WHERE id = o.customer_id) as phone//=>자꾸만보고싶네?
FROM orders o

```

**✅ 한 번에 조인**

```sql

SELECT
  o.customer_id,
  c.name,
  c.email,
  c.phone
FROM orders o
JOIN customers c ON o.customer_id = c.id

```

같은 일을 반복하지 말고 한번에 처리할 수 있는 방법을 찾아보자!
비슷한 문제

```sql
SELECT *
FROM orders
WHERE
  customer_id IN (SELECT id FROM customers WHERE region = 'US')
  AND EXISTS (SELECT 1 FROM customers WHERE id = orders.customer_id AND active = true)

```

customer table 1번 방문~

후 또 방문~

⇒ 개선

```sql
SELECT o.*
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.region = 'US'
  AND c.active = true

```

# 11.3 생각의 전환

## **사용자 함수 최소화**

조인 방식으로 대체하여 성능 향상

- 사용자 정의 함수(UDF)는 편리
- 호출될 때마다 별도의 실행 컨텍스트가 생성되어 오버헤드가 큼
- 안에서 테이블 접근이 있으면 특히 더 느려짐

---

🧐

안에서 테이블 접근이 있으면 왜 특히 더 느려지지?

사용자 정의 함수가 호출될 때 PostgreSQL

1. **별도의 실행 컨텍스트(context)** 생성
2. **함수 안의 쿼리를 독립적으로 실행**
3. **호출 횟수마다 반복 수행**

1️⃣ **함수 내부 쿼리는 메인 쿼리의 실행 계획과 별개다**

```sql
CREATE FUNCTION get_customer_name(customer_id INT)
RETURNS TEXT AS $$
  SELECT name FROM customers WHERE id = customer_id;
$$ LANGUAGE sql;

SELECT order_id, get_customer_name(customer_id)
FROM orders;

```

여기서 PostgreSQL은 `orders` 테이블을 **한 줄 읽을 때마다**

- `get_customer_name()`을 **따로 실행**
- → 즉, `customers` 테이블을 **수천 번 반복해서 쿼리**

**조인 방식이라면 Hash Join 1회**

- 단 1번의 `Hash Join` 또는 `Nested Loop Join`
- customers 테이블은 1회 스캔 또는 인덱스 탐색만

BUT

**함수 방식이라면 N번의 쿼리 호출 = N번의 I/O**

**⇒ 함수 내 테이블 접근은 N+1 쿼리 문제를 유발!!! JPA에서 익히 본..그것..**

- 각 row마다 `SELECT name FROM customers WHERE id = ?`가 실행됨
- → N번 실행 = **N번 테이블 접근**

---

🧐

### `Order` → `Customer`가 ManyToOne일 때

```java

@Entity
class Order {
  @ManyToOne(fetch = FetchType.LAZY)
  private Customer customer;
}

```

```sql
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    System.out.println(order.getCustomer().getName());
}
```

### 내부적으로 발생하는 쿼리

1. `SELECT * FROM orders;` (1번)
2. 이후 반복

```sql
-- order.customer.id가 101
SELECT * FROM customers WHERE id = 101;

-- 다음은 102
SELECT * FROM customers WHERE id = 102;

-- 다음은 103 ...
→ orders 수만큼 N개의 customer 조회 쿼리 발생 = N+1
```

뭔가 근본적으로 같은 문제같다!!

해결 전략도 동일~

| 문제 해결 방식 | PostgreSQL | JPA |
| --- | --- | --- |
| 반복 호출 제거 | **JOIN으로 통합** | **fetch join 사용** |
| 결과 캐싱 | CTE/Temp Table 등 | 2차 캐시, EntityGraph |
| 필터용만 사용할 것 | 조인 유지 | DTO projection |
| 결과 캐싱 | CTE/Temp Table 등 | 2차 캐시, EntityGraph |
| 필터용만 사용할 것 | 조인 유지 | DTO projection |
---

2️⃣ **PostgreSQL은 함수 내부 쿼리를 “미리 최적화”하지 못한다**

- 함수 내부는 외부 쿼리로부터 **블랙박스**처럼 취급됨
- 따라서 PostgreSQL 옵티마이저는
    - 인덱스 전략, 조인 방식, 병렬 처리 등을 적용 못 함
- 결과적으로 **함수 내 테이블 접근은 항상 비효율적인 단일 실행이 됨**
---

```sql
-- 😱 고객명을 가져오는 사용자 함수
CREATE FUNCTION get_customer_name(customer_id INT) 
RETURNS VARCHAR(100)
BEGIN
    DECLARE customer_name VARCHAR(100);
    SELECT name INTO customer_name 
    FROM customers 
    WHERE id = customer_id;
    RETURN customer_name;
END;

-- 😱 주문 목록 조회할 때마다 함수 호출
SELECT 
    order_id,
    customer_id,
    get_customer_name(customer_id) as customer_name,  -- 😱 매번 DB 접근
    order_date,
    amount
FROM orders 
WHERE order_date = '2024-03-15';  -- 1000건이면 함수 1000번 호출!

-- 😱 집계 함수의 남용
CREATE FUNCTION get_customer_total_orders(customer_id INT)
RETURNS INT
BEGIN
    DECLARE total_count INT;
    SELECT COUNT(*) INTO total_count
    FROM orders 
    WHERE customer_id = customer_id;
    RETURN total_count;
END;

-- 😱 고객 목록에서 각자의 주문 수 조회
SELECT 
    customer_id,
    name,
    get_customer_total_orders(customer_id) as total_orders  -- 😱 N+1 문제 발생
FROM customers
LIMIT 100;  -- 100명이면 주문 테이블을 100번 접근!
```

```sql
-- 😊 조인으로 한 번에 처리
SELECT 
    o.order_id,
    o.customer_id,
    c.name as customer_name,  -- 😊 조인으로 한 번에 가져옴
    o.order_date,
    o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.order_date = '2024-03-15';

-- 😊 집계도 서브쿼리나 윈도우 함수로
SELECT 
    c.customer_id,
    c.name,
    COALESCE(o.total_orders, 0) as total_orders
FROM customers c
LEFT JOIN (
    SELECT 
        customer_id,
        COUNT(*) as total_orders
    FROM orders 
    GROUP BY customer_id
) o ON c.customer_id = o.customer_id
LIMIT 100;

-- 😊 또는 윈도우 함수 활용 (고객별 상세 주문 정보와 함께)
SELECT DISTINCT
    c.customer_id,
    c.name,
    COUNT(o.order_id) OVER (PARTITION BY c.customer_id) as total_orders
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```

### 함수를 써도 괜찮을 때

```sql
-- ✅ 계산만 하는 함수 (테이블 접근 없음)
CREATE FUNCTION calculate_tax(amount DECIMAL, rate DECIMAL)
RETURNS DECIMAL
BEGIN
    RETURN amount * rate;
END;

-- ✅ 포맷팅 함수 
CREATE FUNCTION format_phone(phone VARCHAR(20))
RETURNS VARCHAR(20)
BEGIN
    RETURN CONCAT(SUBSTRING(phone,1,3), '-', SUBSTRING(phone,4,4), '-', SUBSTRING(phone,8,4));
END;

-- ✅ 복잡한 비즈니스 로직 (가끔 사용)
CREATE FUNCTION calculate_discount_rate(customer_grade VARCHAR(10), order_amount DECIMAL)
RETURNS DECIMAL
BEGIN
    -- 복잡한 할인율 계산 로직
    -- 테이블 접근 없이 순수 계산만
END;
```

## **작업량 감소**: CASE 문 최적화로 연산 횟수 줄이기

## 계산 미리 하기, 계산량 줄이기

**특히 대용량 데이터 처리 시!!**

**❌ 매번 복잡한 계산**

```sql
-- 300만 건마다 12번 CASE 문 = 3600만 번 연산
SELECT
  customer_id,
  SUM(CASE WHEN order_date LIKE '2024-01%' THEN amount END) as jan,
  SUM(CASE WHEN order_date LIKE '2024-02%' THEN amount END) as feb,
-- ... 12개월 반복
FROM huge_orders
GROUP BY customer_id

```

**✅ 단계별 처리로 계산량 감소**

```sql

-- 1단계: 월별로 미리 집계 (결과 1000건)
WITH monthly_summary AS (
  SELECT
    customer_id,
    DATE_FORMAT(order_date, '%Y-%m') as month,
    SUM(amount) as monthly_amount
  FROM huge_orders
  GROUP BY customer_id, DATE_FORMAT(order_date, '%Y-%m')
)
-- 2단계: 1000건에 대해서만 CASE 문 (1000 * 12 = 12000번 연산)
SELECT
  customer_id,
  SUM(CASE WHEN month = '2024-01' THEN monthly_amount END) as jan,
  SUM(CASE WHEN month = '2024-02' THEN monthly_amount END) as feb
-- ...
FROM monthly_summary
GROUP BY customer_id

```
