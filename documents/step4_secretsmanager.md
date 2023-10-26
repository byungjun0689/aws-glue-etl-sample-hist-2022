## 4. AWS Secrets Manager

### 4.1 생성

- AWS Glue 내 Job Script에 JDBC 관련 정보 노출 최소화를 위한 AWS 내 정보관리서비스 이용.
- URL
    - JDBC 접근 URL 정보 전부가 필요함.
    - `DB 연결시 복사 했던 URL 사용`
- AWS `Secrets Manager` → 새로운 보안 저장 → `다른 유형의 보안 암호` 선택
- 보안 암호 이름 지정 후 저장.
    - 이름 : `hist-retail-postgresql`
    - `db_url`
        - jdbc:postgresql://{endpoint}:5432/postgres
        - jdbc:postgresql://[hist-retail-hands-on-rds.cwt0xlqwsn2i.ap-northeast-2.rds.amazonaws.com](http://hist-retail-hands-on-rds.cwt0xlqwsn2i.ap-northeast-2.rds.amazonaws.com/):5432/postgres
    - `db_user`
        - postgres
    - `db_pw`
        - ******* : DB에서 설정한 PW

![Untitled](../img/Untitled%2018.png)

### 4.2 SecretsManager VPC Endpoint

- `VPC Endpoint` 생성 필요(**Secrets Manager**) → 사용할 VPC 설정 + public Subnet 선택
    - 설정하지 않으면 미국 리전으로 SecretsManager로 연결 시도.
    
    ```
    // 에러 메세지
    
    EndpointConnectionError: Could not connect to the endpoint URL: "https://secretsmanager.ap-northeast-2.amazonaws.com/"
    ```
    
- `VPC 서비스 → 엔드포인트(메뉴) → 엔드포인트 생성(버튼)`
    - 이름 : `{메일id}-handson-sm-vpce`
    - VPC → `{메일id}-handson-vpc` (이전에 생성했던 VPC)
    - 서브넷
        - 사용가능한 서브넷 → `public subnet` 선택
    - 보안그룹
        - `{메일id}-handson-sg` (이전에 생성했던 sg)
    
    ![Untitled](../img/Untitled%2019.png)