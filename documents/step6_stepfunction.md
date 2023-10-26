## 6. Stepfunction - Workflow 생성(Stepfunction → Airflow로 실습 추천)

### 6.1 IAM Role 생성(Stepfunction → Glue Handling)

- IAM → 역할(Role 생성) → 서비스 (Step functions) 선택.
- 명칭
    - {메일id}_stepfunction_glue_role
- Role 생성 이후 권한 추가
    - `AWSGlueConsoleFullAccess`
    - `CloudWatchFullAccess`
        - Stepfunction 로그 확인용.

![Untitled](../img/Untitled%2056.png)

![Untitled](../img/Untitled%2057.png)

![Untitled](../img/Untitled%2058.png)

### 6.2 Stepfunction 생성

- EventBridge → Lambda 를 통해 Stepfunction을 Daily로 수행 할 수 있도록 한다는 가정하에 Stepfunction workflow 생성(실습 미포함)
- [EventBridge → Lambda 예재](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/events/RunLambdaSchedule.html)
    - Amazon EventBridge 서비스 → 이벤트 → 규칙 → 규칙 생성 → 원하는 상세 조건 선택
- Glue 내 Workflow 대신 Stepfunction 을 사용하는 이유
    - Glue 내 WorkFlows는 `파라미터`를 받을 수 없음. → `날짜 별로` 수행을 위해서
- Stepfunction 흐름 도식화

![Untitled](../img/Untitled%2059.png)

- 최종 Stepfunction 흐름.

![Untitled](../img/Untitled%2060.png)

### 생성

![Untitled](../img/Untitled%2061.png)

### 6.1. 병렬(Parallel State) 선택

![Untitled](../img/Untitled%2062.png)

### 6.2. GlueStartJobRun : 발주 데이터 추출 (FROM DB)

![Untitled](../img/Untitled%2063.png)

- API 파라미터
    - 파라미터 값을 Lambda 에서 전달된 파라미터 값으로 실행
        - "--EXTRACT_DATE.$": "$.EXTRACT_DATE"
        - Key.$ : $.Value 와 같이 Key와 Value 앞뒤에 $를 붙여야 변수로 인식하여 값을 전달함.

```json
// Step1-balju
{
  "JobName": "jb_retail_factdata_d2s",
  "Arguments": {
    "--TARGET_DATA": "balju",
    "--EXTRACT_DATE.$": "$.EXTRACT_DATE"
  }
}
//Step1-sales
	{
	  "JobName": "jb_retail_factdata_d2s",
	  "Arguments": {
	    "--TARGET_DATA": "sales",
	    "--EXTRACT_DATE.$": "$.EXTRACT_DATE"
	  }
	}
```

- 테스트 완료 대기 선택.
- **출력 지정**
    - INPUT에서 온 데이터 + 결과 중 `$.Arguments`에 포함된 데이터만 결과로 전달.

![Untitled](../img/Untitled%2064.png)

### 6.2. Start Crawler

![Untitled](../img/Untitled%2065.png)

- 각 Job 에 맞는 Crawler 로 흘러가도록 선택
- Step1-balju
    
    ```json
    {
      "Name": "cr_retail_factdata_balju"
    }
    ```
    
- Step1-sales
    
    ```json
    {
      "Name": "cr_retail_factdata_sales"
    }
    ```
    
- 출력 선택 (둘다 동일)
    - 입력을 그대로 출력되도록 처리.
    
    ![Untitled](../img/Untitled%2066.png)
    
    ### 6.3. GetCrawler
    
    - Crawler 현재 어느 상태인지 확인하는 Step → 결과를 다음 Job 으로 전달하기 위한 단계이기도함.
    - 두개 Crawler 다 동일하게 처리
    - GetCrawler에서 나온 결과를 $.GetCrawler 안으로 넣어서  Output되도록 처리.
    
    ![Untitled](../img/Untitled%2067.png)
    
    ![Untitled](../img/Untitled%2068.png)
    
    - GetCrawler를 실행 → `**결과값**`
    
    ```json
    {
      "--TARGET_DATA": "sales",
      "--EXTRACT_DATE": "2015-01-05",
      "GetCrawler": {
        "Crawler": {
          "Classifiers": [],
          "CrawlElapsedTime": 49000,
          "CreationTime": "2022-08-19T06:46:42Z",
          "DatabaseName": "hist-retail",
          "LakeFormationConfiguration": {
            "AccountId": "",
            "UseLakeFormationCredentials": false
          },
          "LastCrawl": {
            "LogGroup": "/aws-glue/crawlers",
            "LogStream": "cr_retail_factdata_sales",
            "MessagePrefix": "62ef7b9f-e54d-4b54-805c-d29e905ac4e4",
            "StartTime": "2022-08-19T08:01:25Z",
            "Status": "SUCCEEDED"
          },
          "LastUpdated": "2022-08-19T07:12:38Z",
          "LineageConfiguration": {
            "CrawlerLineageSettings": "DISABLE"
          },
          "Name": "cr_retail_factdata_sales",
          "RecrawlPolicy": {
            "RecrawlBehavior": "CRAWL_NEW_FOLDERS_ONLY"
          },
          "Role": "AWSGlueServiceRole-PipelineRole",
          "SchemaChangePolicy": {
            "DeleteBehavior": "LOG",
            "UpdateBehavior": "LOG"
          },
          "State": "STOPPING", // 다음 Step Choice State에서 사용할 Key값.
          "Targets": {
            "CatalogTargets": [],
            "DeltaTargets": [],
            "DynamoDBTargets": [],
            "JdbcTargets": [],
            "MongoDBTargets": [],
            "S3Targets": [
              {
                "Exclusions": [
                  "**/_temporary/**"
                ],
                "Path": "s3://hist-retail/factdata/sales"
              }
            ]
          },
          "Version": 4
        }
      }
    }
    ```
    
    ### 6.4. Choice State
    
    - Crawler의 상태에 따라 Wait를 할 것인지 Success를 진행 할 것인지 판단.
    - Input 값을 그대로 Output 값을 가지고 감.
    
    ![Untitled](../img/Untitled%2069.png)
    
    - Rules
        
        ![Untitled](../img/Untitled%2070.png)
        
        - 전체 식
            - $.GetCrawler.Crawler.State == "RUNNING"
        - `Variable`
            - $.GetCrawler.Crawler.State
        - `Operator`
            - is equal to
        - `Value`
            - String constant
        - `Text`
            - RUNNING
        
        ![Untitled](../img/Untitled%2071.png)
        
    
    ### 6.5. Step 2 StartJobRun
    
    - 단 Step2의 Glue Job에서는 파라미터 값
        - $.extract_date가 아닌 `$.--EXTRACT_DATE`로 지정해야됨.
            - 입력 값 부터 `--EXTRACT_DATE`로 파라미터 형태로 넘어오기 때문에.
    
    ```json
    // balju
    {
      "JobName": "jb_retail_silver_dailybalju_s2s",
      "Arguments": {
        "--EXTRACT_DATE.$": "$.--EXTRACT_DATE"
      }
    }
    //sales
    {
      "JobName": "jb_retail_silver_dailysales_s2s",
      "Arguments": {
        "--EXTRACT_DATE.$": "$.--EXTRACT_DATE"
      }
    }
    ```
    
    - 출력
        - 설정 X → 추후에 파라미터로 처리 하는 부분 없음.
    
    ### 6.6. Step 2 StartCrawler
    
    - Sales의 경우 3개의 결과물이 나오므로 3개의 Crawler를 병렬로 수행 ( Parallel State )
        - 순차가 필요 없어 한꺼번에 실행
    - 각 Crawler 에 맞는 이름만 작성하면 됨. (위 StartCrawler 참조)
    
    ![Untitled](../img/Untitled%2072.png)
    
    ### 6.7. 최종 Step
    
    ![Untitled](../img/Untitled%2060.png)
    
    ### 6.8 이름 지정
    
    - 이름
        - {메일id}_hist_retail_dailybatch
    - 로그 수준
        - ALL → ERROR 로 변경.
    
    ![Untitled](../img/Untitled%2073.png)
    
    ### 6.9 최종 결과 확인
    
    - 실행 시작
    
    ```json
    {
        "EXTRACT_DATE": "2015-01-01"
    }
    ```
    
    ![Untitled](../img/Untitled%2074.png)
    
    - Raw data 생성 여부 확인
        
        ![Untitled](../img/Untitled%2075.png)
        
    - Silver 데이터 생성 여부 확인
        
        ![Untitled](../img/Untitled%2076.png)
        
        ### 6.10 Lambda function 생성
        
        - 역할 IAM 에서 신규 생성.
            - 이름
                - {메일id}-LambdaToStepfunctions
            - 역할 생성 → 서비스 Lambda 선택 -> `AWSStepFunctionsConsoleFullAccess` 추가 생성
            
            ![Untitled](../img/Untitled%2077.png)
            
            ![Untitled](../img/Untitled%2078.png)
            
        - Lambda 함수 생성
            - 함수 이름 : {메일id}-hist-retail-dailybatch
            - 런타임 : Python3.9
            - 역할 LambdaToStepfunctions-Pipeline으로 선택
            
            ![Untitled](../img/Untitled%2079.png)
            
        - Script
            - stateMachineArn : 위에서 생성한 `Stepfunction` Arn 을 복사.
            
            ![Untitled](../img/Untitled%2080.png)
            
            ```python
            import json
            import boto3
            
            def lambda_handler(event, context):
            
                batch_date = '2015-01-05'
                
                client = boto3.client('stepfunctions')
            
                response = client.start_execution(
                    stateMachineArn='aarn:aws:states:ap-northeast-2:363292119458:stateMachine:blee_hist_retail_dailybatch',
                    input='{"EXTRACT_DATE":"'+ batch_date + '"}'
            
                )
                
                # TODO implement
                return {
                    'statusCode': 200
                }
            ```
            
            - 테스트 이벤트 구성 → Deploy
                
                ![Untitled](../img/Untitled%2081.png)
                
        - 테스트
            
            ![Untitled](../img/Untitled%2082.png)