## 1. S3 버킷 및 폴더 생성
💡 `[설명]`
[AWS S3 란?](https://www.notion.so/AWS-S3-8c5afd0c5df64f589009b10de7df1c52?pvs=21)

- `S3` Bucket 생성 및 폴더 생성
    1. Bucket 이름 : `{메일id}-hist-retail`
        1. 동일한 Bucket 이름 지정 불가.
        2. 리전 : `ap-northeast-2(서울)`
        3. 옵션 : 나머지는 변경 사항 없음.
    2. 폴더(최상위 폴더만 생성, temp는 생성필요)
        1. `dimension` : 코드성 데이터를 관리하는 폴더
            1. `products` : 제품 데이터 
            2. 1회만 수행하고 데이터가 추가 됐을때만 수행 하면 됨.
        2. `factdata` : 트랜잭션 데이터를 관리하는 폴더
            1. `sales` : 판매 데이터 일별로 적재 (partition by 날짜)
            2. `balju` : 발주 데이터 일별로 적재 (partition by 날짜)
        3. `silver` : Raw Data 에서 변환된 결과물이 저장되는 폴더.
        4. `temp(필수)`
            1. `tmp` : glue job에서 사용할 temp folder
            2. `script` : glue job script 저장
            3. `logs` : glue job pyspark log, Spark History를 볼 수 있도록 처리하는 폴더.

![Untitled](../img/Untitled 01.png)