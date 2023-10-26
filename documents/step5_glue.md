## 5. Glue

- AWS 개편이 되면서 Glue 서비스 내 Legacy Pages에 모든 메뉴가 들어가게 되면서 해당 Top 메뉴에서 아래 기능들을 생성.


💡 `[설명]`
[AWS Glue란?](https://www.notion.so/AWS-Glue-21a4e620cac84c54a1960d5f7d801697?pvs=21)


### 5.0 Glue IAM 생성

- `IAM(서비스)` 역할 생성 → 서비스 `Glue` 선택
    - Secrets Manager 접근 `SecretsManagerReadWrite` 권한 추가 필요
    - S3 접근 `AmazonS3FullAccess`
    - Glue 서비스를 이용할 수 있는 `AWSGlueServiceRole` 선택
    - `참조`
        - 정책 검색 후 선택 → 필터링 지우기 → 다시 검색 형태로 위 3개 항목 선택.
    - 이름
        - `{메일id}-handson-glue-role`

![Untitled](../img/Untitled%2020.png)

![Untitled](../img/Untitled%2021.png)

### 5.1 연결 추가~~(필요없음)~~

- 데이터를 추출할 DB(Source) 가 Private Subnet에 위치해 있다면 해당 Private Subnet 과 통신이 가능한 Subnet에 Glue Instance를 실행시켜야되는데 그 때 사용하는 기능
- Glue ETL 작업에서 `Glue → Aurora(Postgresql)`에 접근하여 데이터를 가지고 오려면 `JDBC` 연결이 필요.
    - Aurora(PostgreSQL) Write Instance 의 연결할 수 있는 JDBC URL 이 필요.
    - 이름
        - `{메일id}-handson-postgresql-connection`
    
    ![Untitled](../img/Untitled%2022.png)
    
    ![Untitled](../img/Untitled%2023.png)
    

### 5.2 Step 1. Job 1 (Dimension)

- Product 제품 데이터를 Postgresql 에서 추출하여 Parquet 형태로 `Raw Data` S3로 저장.
- Job 명칭
    - `{메일id}_jb_retail_dimension_products_d2s`
    - `규칙` : {메일id}_jb_프로젝트_타입_테이블명_(db : d, 2:to, s : s3)

![Untitled](../img/Untitled%2024.png)

![Untitled](../img/Untitled%2025.png)

![Untitled](../img/Untitled%2026.png)

- Job Python Script
    - outputPath : S3 Path 변경 필요.
        - {bucket_name} ⇒ 본인 S3 Bucket 으로 변경.
    - SecretsManager 명 변경 필요.
        - {secretmanager_name} ⇒ 본인 secretmanager 으로 변경.
    
    ```python
    import sys
    from awsglue.utils import getResolvedOptions
    from pyspark.context import SparkContext
    from awsglue.context import GlueContext
    from awsglue.dynamicframe import DynamicFrame
    from awsglue.job import Job
    import boto3
    from pyspark.sql import functions as F
    import json
    
    # --
    # -- SparkContext 생성
    # --
    glueContext = GlueContext(SparkContext.getOrCreate())
    spark = glueContext.spark_session
    spark.sparkContext.setLogLevel("ERROR")
    
    # --
    # -- Overwrite setting
    # --
    spark.conf.set("spark.sql.sources.partitionOverwriteMode","dynamic")  #  없으면 전체 Partition이 overwrite 된다 
    hadoop_conf = glueContext._jsc.hadoopConfiguration()
    hadoop_conf.set("mapreduce.fileoutputcommitter.marksuccessfuljobs", "false")  # SUCCESS 폴더 생성 방지
    #hadoop_conf.set("fs.s3.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")  # $folder$ 폴더  생성 방지 
    
    # --
    # -- @params: [JOB_NAME]
    # --
    args = getResolvedOptions(sys.argv, ['JOB_NAME'])
    
    job = Job(glueContext)
    job.init(args['JOB_NAME'], args)
    
    bucket_name = 'blee-hist-retail'
    outputPath = f's3a://{bucket_name}/dimension/products'
    
    #secretClient = boto3.client("secretsmanager", region_name="ap-northeast-2")
    secretClient = boto3.client("secretsmanager")
    
    secretmanager_name = "blee-handson-postgresql-sm2"
    get_secret_value_response = secretClient.get_secret_value(SecretId=f"{secretmanager_name}")
    secret = get_secret_value_response['SecretString']
    secret = json.loads(secret)
    
    db_username = secret.get('db_user')
    db_password = secret.get('db_pw')
    db_url = secret.get('db_url')
    
    pushdownquery = "SELECT * FROM \"RETAIL\".\"products\""
    df = spark.read.format("jdbc") \
        .option("url",db_url) \
        .option("dbtable",  "\"RETAIL\".\"products\"") \
        .option("user",db_username) \
        .option("password",db_password) \
        .option("driver","org.postgresql.Driver").load()
    
    if df.count() > 0:
        # S3로 적재
        df.write.mode('overwrite').parquet(outputPath)
    ```
    
- 결과
    
    ![Untitled](../img/Untitled%2027.png)
    
    ![Untitled](../img/Untitled%2028.png)
    

### 5.3 Step 1. Job 2 ( Fact Data, 구매, 발주 데이터)

- 판매, 발주(폐기, 환불) 데이터를 Postgresql 에서 추출하여 Parquet 형태로 `RawData` S3로 저장.
- 속성
    - Job 명칭
        - {메일id}_jb_hist_retail_factdata_d2s
    - `최대 동시성` : 2
        - StepFunction에서 2개 job을 동시에 돌리기 위해서.
    - `파라미터`
        - 파라미터 적용시  “--파라미터명” 으로 -- 가 필수로 들어가야 적용 됨.
        - EXTRACT_DATE
            - 날짜 (yyyy-MM-dd)
                - 예) 2015-01-02
        - TARGET_DATA
            - 고정 String 값 2개
                - balju (발주데이터)
                - sales (거래데이터)
    - 나머지는 첫번째 Job 생성과 동일
- `테스트`
    - `Job 2회 실행`
    - 각 실행 마다 Job 편집을 통해 `TARGET_DATA` 값을 변경. EXTRACT_DATE 값은 고정.
        - balju
        - sales
- 생성 방법은 위와 동일, 단 파라미터에만 아래와 같이 적용.
    - 보안 구성, 스크립트 라이브러리 및 작업 파라미터(선택항목) 메뉴에서
        
        ![Untitled](../img/Untitled%2029.png)
        
- 연결
    
    ![Untitled](../img/Untitled%2026.png)
    
- 스크립트
    - outputPath : S3 Path 변경 필요.
        - {bucket_name} ⇒ 본인 S3 Bucket 으로 변경.
    - SecretsManager 명 변경 필요.
        - {secretmanager_name} ⇒ 본인 secretmanager 으로 변경.
- 결과
    
    ```python
    import sys
    from awsglue.utils import getResolvedOptions
    from pyspark.context import SparkContext
    from awsglue.context import GlueContext
    from awsglue.dynamicframe import DynamicFrame
    from awsglue.job import Job
    import boto3
    from pyspark.sql import functions as F
    import json
    
    # --
    # -- SparkContext 생성
    # --
    glueContext = GlueContext(SparkContext.getOrCreate())
    spark = glueContext.spark_session
    spark.sparkContext.setLogLevel("ERROR")
    
    # --
    # -- Overwrite setting
    # --
    spark.conf.set("spark.sql.sources.partitionOverwriteMode","dynamic")  #  없으면 전체 Partition이 overwrite 된다 
    hadoop_conf = glueContext._jsc.hadoopConfiguration()
    hadoop_conf.set("mapreduce.fileoutputcommitter.marksuccessfuljobs", "false")  # SUCCESS 폴더 생성 방지
    hadoop_conf.set("fs.s3.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")  # $folder$ 폴더  생성 방지 
    
    # --
    # -- @params: [JOB_NAME]
    # --
    args = getResolvedOptions(sys.argv, ['JOB_NAME','EXTRACT_DATE','TARGET_DATA'])
    
    job = Job(glueContext)
    job.init(args['JOB_NAME'], args)
    
    extract_date = args['EXTRACT_DATE']
    target_data = args['TARGET_DATA']
    
    bucket_name = 'blee-hist-retail'
    outputPath = f's3a://{bucket_name}/factdata/{target_data}'
    
    secretmanager_name = "blee-handson-postgresql-sm2" # 이름 변경 필요.
    secretClient = boto3.client("secretsmanager", region_name="ap-northeast-2")
    get_secret_value_response = secretClient.get_secret_value(SecretId=f"{secretmanager_name}")
    secret = get_secret_value_response['SecretString']
    secret = json.loads(secret)
    
    db_username = secret.get('db_user')
    db_password = secret.get('db_pw')
    db_url = secret.get('db_url')
    
    if target_data == "balju":
        db_table = "balju_refund"
        where_field = "balju_date"
    else:
        db_table = "transaction_order"
        where_field = "tr_date"
        extract_date = extract_date.replace('-','')
        
    pushdownquery = "SELECT * FROM \"RETAIL\".\"{db_table}\" WHERE {where_field} = '{extract_date}'".format(extract_date=extract_date, db_table=db_table, where_field=where_field)
    
    df = spark.read.format("jdbc") \
        .option("url",db_url) \
        .option("query",pushdownquery) \
        .option("user",db_username) \
        .option("password",db_password) \
        .option("driver","org.postgresql.Driver").load()
    
    if df.count() > 0:
        # S3로 적재 
        df.write.mode('append').partitionBy(where_field).parquet(outputPath)
    ```
    

![Untitled](../img/Untitled%2030.png)

![Untitled](../img/Untitled%2031.png)

### 5.4 Step 1. Crawler (Step 1, Dimension/Factdata) 생성

- **`5.4.1 Dimention Product Data Crawler 생성`**
    - Crawler 명칭
        - `규칙` : {메일id}_cr_프로젝트_타입_테이블명
        - blee_cr_hist_retail_dimension_products
    - `**필수**`
        - 제외 → `**패턴 추가**` (_temporary 폴더가 생기면서 Table이 꼬인다)
        - ****/_temporary/****
    - PATH
        - s3://hist-retail/dimension/products
        - 해당 폴더까지 지정.
    
    ![Untitled](../img/Untitled%2032.png)
    
    ![Untitled](../img/Untitled%2033.png)
    
    ![Untitled](../img/Untitled%2034.png)
    
    - 제외 `**패턴 추가**` (_temporary 폴더가 생기면서 Table이 꼬인다)
        - ****/_temporary/****
    
    ![Untitled](../img/Untitled%2035.png)
    
    - 크롤러 실행
    
    ![Untitled](../img/Untitled%2036.png)
    

- `**5.4.2 Fact Data Crawler 생성 (발주,거래)**`
    - 발주와 거래 데이터는 같은 S3 폴더에 위치하므로 한꺼번에 수행 가능.
    - 명칭 : {메일id}_cr_hist_retail_factdata_sales, balju
        - 총 2개 크롤러 생성.
    - `**필수**`
        - 제외 → `**패턴 추가**` (_temporary 폴더가 생기면서 Table이 꼬인다)
        - ****/_temporary/****
    
    ![Untitled](../img/Untitled%2037.png)
    
    - 포함 경로
        - s3://blee-hist-retail/factdata/sales
        - s3://blee-hist-retail/factdata/balju
        - 사실 아래 처럼 진행해도 무관하나 Stepfunction 구성 부분에서 2개로 나눠서 개발하는 것으로 구성되어 2개로 나누는 것을 추천드립니다.
    
    ![Untitled](../img/Untitled%2038.png)
    
- 결과
    
    ![Untitled](../img/Untitled%2039.png)
    
- Athena 데이터 조회
    - 필요시 Athena 를 통해 ad-hoc 분석을 수행하여 Data Market 구성을 어떻게 할지 결정.
    
    ![Untitled](../img/Untitled%2040.png)
    

### 5.5 Step 2. Silver Data 생성.

- 데이터 생성 기준(Silver)
    - 거래 데이터
        - 일자별 매출 / 거래 수
        - 일자별 시간별 매출 / 거래 수
        - 일자별 시간별 소분류별 매출 / 거래 수
        - 일자별 시간별 제품(상세)별 매출 / 거래 수
    - 발주 데이터
        - 일자별 소분류별 발주 / 폐기 / 환불 수
- 각 항목에 맞도록 Job, Crawler 생성
    - Job : 발주, 매출 2개 생성
    - Crawler 4개 생성
        - 발주 1개(일별 발주)
            - blee_cr_hist_retail_silver_balju
        - 매출 1개(일시간별 매출, 일시간카테고리별 매출, 일시간상세제품별 매출)
            - blee_cr_hist_retail_silver_sales
            - 1개의 크롤러로 3개 Table 크롤링.

### 5.6 Step 2. Job 생성 ( 발주 데이터 )

- 명칭
    - blee_jb_hist_retail_silver_dailybalju_s2s
- 속성
    - `파라미터`
        - EXTRACT_DATE
            - 날짜 (2015-01-01)
    - 연결
        - 선택 필요 없음
        - PostgreSQL 접근이 없기 때문에 위에 생성했던 Job과 다르게 연결을 선택할 필요 없음.
- 화면 캡처
    
    ![Untitled](../img/Untitled%2041.png)
    
    ![Untitled](../img/Untitled%2042.png)
    
- **Script**
    - `bucket_name 변경 필요.` ⇒ 본인 S3 Bucket 으로 변경.

```python
import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.dynamicframe import DynamicFrame
from awsglue.job import Job
import boto3
from pyspark.sql import functions as func
import json

from botocore.exceptions import ClientError
from pyspark.sql.functions import sum

args = getResolvedOptions(sys.argv, ['JOB_NAME','EXTRACT_DATE'])

bucket_name = 'blee-hist-retail' # 이름 변경 필요.
outputPath = f"s3a://{bucket_name}/silver/dailybalju"

# --
# -- SparkContext 생성
# --
glueContext = GlueContext(SparkContext.getOrCreate())
spark = glueContext.spark_session
spark.sparkContext.setLogLevel("ERROR")

spark.conf.set("spark.sql.sources.partitionOverwriteMode","dynamic")  #  없으면 전체 Partition이 overwrite 된다 
hadoop_conf = glueContext._jsc.hadoopConfiguration()
hadoop_conf.set("mapreduce.fileoutputcommitter.marksuccessfuljobs", "false")  # SUCCESS 폴더 생성 방지
hadoop_conf.set("fs.s3.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")  # $folder$ 폴더  생성 방지 

job = Job(glueContext)
job.init(args['JOB_NAME'], args)

product_dynamic_frame = glueContext.create_dynamic_frame.from_catalog(database = 'hist-retail', table_name = 'products')
product_df = product_dynamic_frame.toDF()

product_category_df = product_df.drop("product_cd").drop("product_name").drop_duplicates()

etl_dt = args['EXTRACT_DATE']

balju_dynamic_frame = glueContext.create_dynamic_frame.from_catalog(database = 'hist-retail', table_name = 'balju', push_down_predicate = "(balju_date in ('{}'))".format(etl_dt))
balju_df = balju_dynamic_frame.toDF()

balju_df2 = balju_df.withColumn("balju_qty", balju_df.balju_qty.cast('int')).withColumn("refund_qty", balju_df.refund_qty.cast('int')).withColumn('disposal_qty', balju_df.disposal_qty.cast('int'))
integ_df = balju_df2.join(product_df, balju_df.product_cd == product_df.product_cd, "inner").drop(product_df.product_cd)

daily_summary_of_balju_df = integ_df.groupBy(["balju_date","subcategory_cd"]).agg(sum("balju_qty").alias("sum_balju"),sum("refund_qty").alias("sum_refund"),sum("disposal_qty").alias("sum_disposal"))

daily_summary_of_balju_df2 = daily_summary_of_balju_df.sort(daily_summary_of_balju_df.balju_date, daily_summary_of_balju_df.subcategory_cd). \
                                join(product_category_df,daily_summary_of_balju_df.subcategory_cd == product_category_df.subcategory_cd, how="INNER"). \
                                drop(daily_summary_of_balju_df.subcategory_cd)

if daily_summary_of_balju_df2.count() > 0:
    daily_summary_of_balju_df2.write.mode('append').partitionBy("balju_date").parquet(outputPath)
```

![Untitled](../img/Untitled%2043.png)

### 5.7 Step 2. Job 생성 ( 매출 데이터 )

- 명칭
    - {메일id}_jb_hist_retail_silver_dailysales_s2s
- 속성
    - `파라미터`
        - EXTRACT_DATE
            - 날짜 (2015-01-01)
    - 연결
        - 선택 필요 없음
        - PostgreSQL 접근이 없기 때문에 위에 생성했던 Job과 다르게 연결을 선택할 필요 없음
- 화면 캡처
    
    ![Untitled](../img/Untitled%2044.png)
    
    ![Untitled](../img/Untitled%2045.png)
    
- Script

```python
import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.dynamicframe import DynamicFrame
from awsglue.job import Job
import boto3
from pyspark.sql import functions as func
import json

from botocore.exceptions import ClientError
from pyspark.sql.functions import sum, avg, count

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'EXTRACT_DATE'])

bucket_name = 'blee-hist-retail'
outputPath = f"s3a://{bucket_name}/silver/sales"

# --
# -- SparkContext 생성
# --
glueContext = GlueContext(SparkContext.getOrCreate())
spark = glueContext.spark_session
spark.sparkContext.setLogLevel("ERROR")

spark.conf.set("spark.sql.sources.partitionOverwriteMode","dynamic")  #  없으면 전체 Partition이 overwrite 된다 
hadoop_conf = glueContext._jsc.hadoopConfiguration()
hadoop_conf.set("mapreduce.fileoutputcommitter.marksuccessfuljobs", "false")  # SUCCESS 폴더 생성 방지
hadoop_conf.set("fs.s3.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")  # $folder$ 폴더  생성 방지 

job = Job(glueContext)
job.init(args['JOB_NAME'], args)

product_dynamic_frame = glueContext.create_dynamic_frame.from_catalog(database = 'hist-retail', table_name = 'products')
product_df = product_dynamic_frame.toDF()

product_category_df = product_df.drop("product_cd").drop("product_name").drop("is_pb").drop_duplicates()
#product_category_df.show(3)

etl_dt = args['EXTRACT_DATE']
order_dynamic_frame = glueContext.create_dynamic_frame.from_catalog(database = 'hist-retail', table_name = 'sales', push_down_predicate = "(tr_date in ('{}'))".format(etl_dt))
order_df = order_dynamic_frame.toDF()

order_df2 = order_df.withColumn('tr_time',order_df.tr_time.cast('int'))
order_df3 = order_df2.join(product_df, order_df2.product_cd == product_df.product_cd, how="inner").drop(order_df2.product_cd)

# Daily + Time 매출
datetime_sales_df = order_df3.groupby(["tr_date","tr_time"]).agg(sum("mount").alias("sum_sales"),count("mount").alias("count_sales")).sort(func.col("tr_time"))
datetime_sales_df.write.mode('append').partitionBy("tr_date").parquet(outputPath+"/dailytime")

# Daily + Time + 소분류별 매출
datetime_cate_sales_df = order_df3.groupby(["tr_date","tr_time","subcategory_cd"]).agg(sum("mount").alias("sum_sales"),count("mount").alias("count_sales")).sort(func.col("tr_time"),func.col("sum_sales").desc())
datetime_cate_sales_df2 = datetime_cate_sales_df.join(product_category_df, datetime_cate_sales_df.subcategory_cd == product_category_df.subcategory_cd, how="left").drop(datetime_cate_sales_df.subcategory_cd)
datetime_cate_sales_df2.write.mode('append').partitionBy("tr_date").parquet(outputPath+"/dailytimeSubcate")

# Daily + Timely + 상세 제품별 매출
datetime_product_sales_df = order_df3.groupby(["tr_date","tr_time","product_cd"]).agg(sum("mount").alias("sum_sales"),count("mount").alias("count_sales")).sort(func.col("tr_time"),func.col("sum_sales").desc())
datetime_product_sales_df2 = datetime_product_sales_df.join(product_df, datetime_product_sales_df.product_cd == product_df.product_cd, how="left").drop(datetime_product_sales_df.product_cd)
datetime_product_sales_df2.write.mode('append').partitionBy("tr_date").parquet(outputPath+"/dailytimeProduct")
```

![Untitled](../img/Untitled%2046.png)

### 5.8 Step 2. Crawler 생성

- 4가지 생성
    - `발주`
        - 이름 : blee_cr_hist_retail_silver_balju
        - 포함 경로 : s3://blee-hist-retail/silver/dailybalju/
    - `판매(3개)`
        - 일별시간별 매출
            - 이름 : blee_cr_hist_retail_silver_sales_dailytime
            - 포함 경로 : s3://blee-hist-retail/silver/sales/dailytime/
        - 일별시간별소분류별 매출
            - 이름 : blee_cr_hist_retail_silver_sales_dailysubcate
            - 포함 경로 : s3://blee-hist-retail/silver/sales/dailytimeSubcate/
        - 일별시간별제품별 매출
            - 이름 : blee_cr_hist_retail_silver_sales_dailyproduct
            - 포함 경로 : s3://blee-hist-retail/silver/sales/dailytimeProduct/
- 발주 Crawler 생성
    - `**필수**`
        - 제외 → `**패턴 추가**` (_temporary 폴더가 생기면서 Table이 꼬인다)
        - ****/_temporary/****
    - IAM 역할
        - blee-handson-glue-role
        - 이전에 생성했던 Role
    - 화면 캡처
        
        ![Untitled](../img/Untitled%2047.png)
        
        ![Untitled](../img/Untitled%2048.png)
        
        ![Untitled](../img/Untitled%2049.png)
        
        ![Untitled](../img/Untitled%2050.png)
        
- 판매 Crawler 생성
    - (Legacy Page) 각각 위에 작성된 경로 3개 발주와 동일한 방법으로 생성
    - (신규페이지) 하나의 크롤러로 3개 테이블 동시 크롤링 가능
        - `(옵션)판매 Crawler 생성`
            - `신규 Crawler 메뉴 선택`
            - Create Crawler
            
            ![Untitled](../img/Untitled%2051.png)
            
            - Add a data source → Create Crawler
                - 일별시간별 매출
                    - 포함 경로 : s3://blee-hist-retail/silver/sales/dailytime/
                - 일별시간별소분류별 매출
                    - 포함 경로 : s3://blee-hist-retail/silver/sales/dailytimeSubcate/
                - 일별시간별제품별 매출
                    - 포함 경로 : s3://blee-hist-retail/silver/sales/dailytimeProduct/
                - 아래와 같이 3개 항목에 대해서 Add a Datasource
                
                ![Untitled](../img/Untitled%2052.png)
                
                ![Untitled](../img/Untitled%2053.png)
                
                ![Untitled](../img/Untitled%2054.png)
                
            - 제외 → `**패턴 추가**` (_temporary 폴더가 생기면서 Table이 꼬인다)
            - ****/_temporary/****
- 결과

![Untitled](../img/Untitled%2055.png)