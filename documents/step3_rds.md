## 3. RDS 생성 및 Import Data to Aurora(`PostgreSql`)

💡 `[설명]`
[AWS RDS](https://www.notion.so/AWS-RDS-24f9127c3f9f4236a19d5b9a5bd96bde?pvs=21)


### 3.1 DB Subnet Group 생성(DB 서브넷 그룹 생성)

- `RDS 서비스 → 서브넷 그룹(메뉴) → DB 서브넷 그룹 생성`
- 이름
    - `{메일id}-db-subnet-group`
- 이름 지정 후 위에서 만든 VPC 선택
- 가용영역 선택 후 : ap-northeast-2a, 2b
- 서브넷
    - Public Subnet으로 선택 → VPC Subnet 메뉴에서 Subnet ID 또는 IP 숙지 하여 선택. → 보통 네트워크 IP 숫자가 작은 것이 Public
        - 10.0.0.0/20
        - 10.0.16.0/20
    - 확인방법
        - VPC 서비스 → 서브넷 → 검색 → {메일id}-handson-subnet-public → IP확인
        
        ![Untitled](../img/Untitled 09.png)
        

![Untitled](../img/Untitled 010.png)

### 3.2 RDS 생성

- AWS 에서 RDS → Aurora(Postgresql) 선택
    - PostgreSQL로 선택해도 무관함.
    - 생성 방법은 동일.
- VPC → 위에서 생성한 `DB Subnet Group` 선택.
- Security Group : 2번에서 생성한 SG 선택.
- 클러스터 이름
    - `{메일id}-handson-postgresql`
- 마스터 암호 : *******(원하는 비밀번호)
- 모니터링 기능 전부 `제거`
    - 운영상에는 필요하겠지만 현재 개발 상태에서는 필요 없음.

![Untitled](../img/Untitled 011.png)

![Untitled](../img/Untitled 012.png)

![Untitled](../img/Untitled 013.png)

### 3.3 DBMS 접근

- DataSouce : `PostgreSQL 선택`
- Aurora Write Instance 앤드포인트를 `Host` 에 입력.
- ID / PW 입력 후 Test Connection 수행.
    - ID : postgres
    - PW : ********
- 참고
    - `Host 입력 후 JDBC 연결 URL Notepad로 복사 → 추후 사용될 예정.`

![Untitled](../img/Untitled 014.png)

### 3.4 스크립트(Create Table)

- DB 생성 및 DBMS 접속 후 아래 Script 를 수행
- Script

```sql
// 기본 postgres DB안에서 스키마 생성 -> Tables 생성.

CREATE SCHEMA "RETAIL";

CREATE TABLE "RETAIL"."products" (
  "product_cd" varchar(20) PRIMARY KEY NOT NULL,
  "product_name" varchar(50) NOT NULL,
  "division_cd" varchar(10) NOT NULL,
  "division_name" varchar(50) NOT NULL,
  "maincategory_cd" varchar(10) NOT NULL,
  "maincategory_name" varchar(50) NOT NULL,
  "subcategory_cd" varchar(10) NOT NULL,
  "subcategory_name" varchar(50) NOT NULL,
  "is_pb" varchar(10)
);

CREATE TABLE "RETAIL"."transaction_order" (
  "tr_date" date NOT NULL,
  "tr_time" varchar(10) NOT NULL,
  "store_cd" varchar(10) NOT NULL,
  "store_name" varchar(20),
  "pos_num" varchar(10),
  "receipt_num" varchar(20) NOT NULL,
  "product_cd" varchar(20) NOT NULL,
  "qty" int,
  "mount" double precision,
  PRIMARY KEY ("tr_date", "tr_time", "pos_num", "receipt_num", "product_cd")
);

CREATE TABLE "RETAIL"."balju_refund" (
  "balju_date" date NOT NULL,
  "product_cd" varchar(20) NOT NULL,
  "balju_qty" int,
  "refund_qty" int,
  "disposal_qty" int,
  PRIMARY KEY ("balju_date", "product_cd")
);

CREATE INDEX ON "RETAIL"."transaction_order" ("tr_date", "tr_time", "receipt_num");

ALTER TABLE "RETAIL"."transaction_order" ADD FOREIGN KEY ("product_cd") REFERENCES "RETAIL"."products" ("product_cd");

ALTER TABLE "RETAIL"."balju_refund" ADD FOREIGN KEY ("product_cd") REFERENCES "RETAIL"."products" ("product_cd");
```

- 위 DB, Table 생성 후 제공된 데이터를 Import Data 수행.
    - products → transaction_order, balju_refund 순으로 실행.
        - transaction_order과 balju_refund 은 `동시 진행`
    - 해당 테이블 선택 후 우클릭 → 데이터 가져오기
    
    ![Untitled](../img/Untitled 015.png)
    
    ![Untitled](../img/Untitled 016.png)
    
    ![Untitled](../img/Untitled 017.png)
    
    - 다음 → 다음 → 진행
- DBeaver 또는 사용하고 있는 DBMS Application을 활용.