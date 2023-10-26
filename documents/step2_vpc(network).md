## 2. VPC 생성 및 Security Group Inbound 규칙 추가

💡 `[설명]`
[VPC, EndPoint, Security Group란?](https://www.notion.so/VPC-EndPoint-Security-Group-c5539bd8b3cb4749949cb035bf96134c?pvs=21)


### 2.1 `VPC` 생성

- VPC 등을 선택 → VPC 이름 입력 후 나머지는 `디폴트 옵션` 그대로 유지
    - 이름 : `{메일id}-handson`
    - Subnet, S3 Endpoint 생성
    
    ![Untitled](../img/Untitled 02.png)
    
- VPC 서비스 → 앤드포인트(Endpoint) → 내 VPC S3 엔드포인트 선택 → 라우팅 테이블 관리 (필요없음)
    - `Public Subnet` → 라우트 테이블에서 체크 해줘야 함.
    - 왜?
        - RDS(DB)를 Public Subnet 에 위치하여 작업을 수행할 예정이라 public subnet에 endpoint 연결이 필요.
        - **AWS Glue 에서 RDS 데이터를 S3로 ETL작업 수행 시 접근 권한이 필요. → 에러 발생.**
    
    ![Untitled](../img/Untitled 03.png)
    
- **`참고 : VPC S3 Endpoint 생성이 안됐을 경우.`**
    - 원래는 VPC 생성하게 될 때 자동으로 생성됨
    - 이미 있을 가능성이 높으나, 필요하다면 아래와 같은 작업이 필요.
    - VPC → 엔드포인트 메뉴 선택
    - 서비스에서 S3검색후 Gateway로 되어있는 부분을 선택.
        - 해당 VPC 선택 후 나머지는 Default로 세팅
    
    ![Untitled](../img/Untitled 04.png)
    

### 2.2 `Security Group` 생성

- VPC 서비스 → 보안 그룹(Security Group) → 보안 그룹 생성

![Untitled](../img/Untitled 05.png)

- 신규 Security Group 생성 후 위 `생성한 VPC`와 연결.
    - 이름
        - `{메일id}-handson-sg`
    - 이름과 VPC만 선택 후 `생성`
    
    ![Untitled](../img/Untitled 06.png)
    

### 2.3 Security Group `Inbound 규칙 추가`

- Security Group 생성 후 Inbound 규칙 추가.
- 규칙 추가 후 비고에 알아 볼 수 있도록 메모 → 추후 편의성 제고.
- `규칙`
    - `PostgreSQL`
        - 유형 : PostgreSQL(5432)
        - 소스 : My IP
        - 설명 : PostgreSQL
    - `AWS Glue(필요없음)`
        - 유형 : HTTPS
        - 소스 : **pl-78a54011 (s3 관리형 접두사)**
        - 설명 : Glue to PostgreSQL
        - `참조` : Glue 서비스 이용을 위한 아래 작업 수행 필요([AWS 문서](https://docs.aws.amazon.com/ko_kr/glue/latest/dg/setup-vpc-for-glue-access.html))
            - Glue → S3 를 사용하기 위해 Https → S3 Endpoint Rule을 규칙으로 추가 해줘야한다.
            - S3 EndPoint : *`s3-prefix-list-id` 가 입력 되어야 함*
                - *`s3-prefix-list-id`* 확인 방법 : `VPC 서비스 - 관리형 접두사 목록`에서 확인 가능하다.
                - 신규 탭 또는 Ctrl(Command) + VPC 서비스 선택 → 목록 확인.
            - 접두사 목록 이름 : com.amazonaws.ap-northeast-2.s3 찾아서 ID를 복사
                - **pl-78a54011**
            
            ![Untitled](../img/Untitled 07.png)
            
    - `자체 통신`
        - 유형 - 모든 TCP
        - 소스 - 현재 본인 Security Group(이름을 타이핑하면 자동적으로 Suggestion 됨)
        - 설명 - Self inbound
        - 참조 - 모든 TCP → 해당 Security group으로 접근 되도록 추가. ( Glue job 실행시 필요, AWS안내 참조)
    
    ![Untitled](../img/Untitled 08.png)