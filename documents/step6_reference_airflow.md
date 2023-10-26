
## [참고] Airflow - Workflow 생성

### [옵션] 설치 및 DB(PostgreSQL) 설정, 병렬처리를 위한 필수 설치.

- 기본적으로 sqlite를 사용하는데 sqlite를 사용할 경우 task를 병렬로 처리할 수 없다.
- 참조
    
    [[airflow] 3. LocalExecutor 사용하기](http://sanghun.xyz/2017/12/airflow-3.-localexecutor-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0/)
    
- sqlite 를 DB로 한다면 `SequentialExecutor`로만 설정이 가능하다
- P**arallel 하게 실행하려면 다른 Executor로 실행해야된다.**
    - `LocalExecutor`나 `CeleryExecutor`
    - 다른 DB가 필요하다.

```bash
brew install postgresql@13 # Version을 적어야 설치가 된다.
pip install psycopg2-binary #airflow와 연결하려면 필요함

echo 'export PATH="/opt/homebrew/opt/postgresql@13/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

brew services restart postgresql@13
```

![Untitled](../img/Untitled%2083.png)

### 연결 및 DataBase 생성

- `psql`명령어를 통해 postgres terminal 접속.
- airflow용 계정과 database를 만든 뒤, 계정에 db접근권한을 부여한다.
- 테이블 권한을 부여하기 위해선, 해당 테이블과 연결된 상태여야 한다. (\c 명령어를 이용)

```bash
psql postgres

postgres=# CREATE DATABASE airflow;
postgres=# CREATE USER blee with ENCRYPTED PASSWORD 'blee';
postgres=# GRANT all privileges on DATABASE airflow to blee;
postgres=# \c airflow
postgres=# grant all privileges on all tables in schema public to blee;

-- Format
-- CREATE DATABASE {database_name};
-- CREATE USER {user_name} with ENCRYPTED PASSWORD '{password}';
-- GRANT all privileges on DATABASE {database_name} to {user_name};
-- \c {database_name}
-- grant all privileges on all tables in schema public to {user_name};
```

### Airflow 설치

```bash
pip install apache-airflow 
pip install 'apache-airflow[amazon]' # amazon aws operator 설치

airflow db init
```

- 설치 후 기본 airflow folder 가 생성됨.

```bash
cd ~/airflow

# on airflow folder
mkdir dags && cd dags
```

### Airflow 실행

```bash
airflow standalone # admin , pw : cli에 표시됨.
```

### 코드

- 파일명 : 아무렇게 해도 상관없음, glue-pipeline.py
- 변경이 필요한 부분
    - job_name
    - role_arn
    - crawler_name
    - aws_conn_id
        
        [AWS 명령줄 인터페이스](https://aws.amazon.com/ko/cli/)
        
        - AWS Cli 설치 후 현재 사용하고 있는 Configuration Profile 이름

```bash
from airflow import DAG
from datetime import datetime
from airflow.providers.amazon.aws.operators.glue_crawler import GlueCrawlerOperator
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator

default_args = {
    'start_date' : datetime.now().strftime("%y-%m-%d")
}

with DAG(
    dag_id='glue-pipeline',
    schedule_interval='@daily',
    default_args=default_args,
    tags=['glue_sample'],
    catchup=False
) as dag:

    job_name = "blee_jb_hist_retail_dimension_products_d2s"
    role_arn = "aws glue 사용 role arn"
	
		# Step 0 Dimension Data(Product) 
    job_step1_products = GlueJobOperator(
        task_id="job_step1_products",
        job_name=job_name,
        job_desc="products(dimenstion)",
        iam_role_name=role_arn,
        aws_conn_id="AWS_DEFAULT_PROFILE"
    )

    glue_crawler_dimenstion_config = {
        "Name": "blee_cr_hist_retail_dimension_products",
        "Role": role_arn,
    }

    crawler_step1 = GlueCrawlerOperator(
        task_id="crawler_step1",
        config=glue_crawler_dimenstion_config,
        aws_conn_id="AWS_DEFAULT_PROFILE")

    #Step 1, Sales Job
    extract_date = '2015-01-14'
    step1_job_name = "blee_jb_hist_retail_factdata_d2s"
    step1_job_sales_script_args = {
        "--EXTRACT_DATE" : extract_date,
        "--TARGET_DATA" : "sales"
    }

    job_step1_fact_sales = GlueJobOperator(
        task_id="job_step1_fact_sales",
        job_name=step1_job_name,
        job_desc="step1-sales",
        iam_role_name=role_arn,
        aws_conn_id="AWS_DEFAULT_PROFILE",
        script_args=step1_job_sales_script_args
    )
    # Step 1. balju Job
    step1_job_balju_script_args = {
        "--EXTRACT_DATE" : extract_date,
        "--TARGET_DATA" : "balju"
    }

    job_step1_fact_balju = GlueJobOperator(
        task_id="job_step1_fact_balju",
        job_name=step1_job_name,
        job_desc="step1-balju",
        iam_role_name=role_arn,
        aws_conn_id="AWS_DEFAULT_PROFILE",
        script_args=step1_job_balju_script_args
    )

    glue_crawler_factdata_config = {
        "Name": "blee_cr_hist_retail_factdata",
        "Role": role_arn,
    }

    crawler_step1_factdata = GlueCrawlerOperator(
        task_id="crawler_step1_factdata",
        config=glue_crawler_factdata_config,
        aws_conn_id="AWS_DEFAULT_PROFILE")

    # Step 2 발주, 매출 데이터 Silver Data 
    script_step2_args = {
        "--EXTRACT_DATE" : extract_date,
    }

    job_step2_fact_balju_name = "blee_jb_hist_retail_silver_dailybalju_s2s"
    job_step2_fact_balju = GlueJobOperator(
        task_id="job_step2_fact_balju",
        job_name=job_step2_fact_balju_name,
        job_desc="step2-balju",
        iam_role_name=role_arn,
        wait_for_completion=True,
        aws_conn_id="AWS_DEFAULT_PROFILE",
        script_args=script_step2_args
    )

    job_step2_fact_sales_name = "blee_jb_hist_retail_silver_dailysales_s2s"
    job_step2_fact_sales = GlueJobOperator(
        task_id="job_step2_fact_sales",
        job_name=job_step2_fact_sales_name,
        job_desc="step2-sales",
        iam_role_name=role_arn,
        wait_for_completion=True,
        aws_conn_id="AWS_DEFAULT_PROFILE",
        script_args=script_step2_args
    )

    # Step 2 crawler
    glue_crawler_step2_balju_config = {
        "Name": "blee_cr_hist_retail_silver_balju",
        "Role": role_arn,
    }

    crawler_step2_balju = GlueCrawlerOperator(
        task_id="crawler_step2_balju",
        config=glue_crawler_step2_balju_config,
        aws_conn_id="AWS_DEFAULT_PROFILE")

    glue_crawler_step2_sales_config = {
        "Name": "blee_cr_hist_retail_silver_sales",
        "Role": role_arn,
    }

    crawler_step2_sales = GlueCrawlerOperator(
        task_id="crawler_step2_sales",
        config=glue_crawler_step2_sales_config,
        aws_conn_id="AWS_DEFAULT_PROFILE")
        
    job_step1_products >> crawler_step1 >> [job_step1_fact_sales, job_step1_fact_balju] >> crawler_step1_factdata
    crawler_step1_factdata >> job_step2_fact_balju>>crawler_step2_balju
    crawler_step1_factdata >> job_step2_fact_sales>>crawler_step2_sales
```

### 결과

![Untitled](../img/Untitled%2084.png)

![Untitled](../img/Untitled%2085.png)