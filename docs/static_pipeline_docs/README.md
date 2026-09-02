# Static Crawling Pipeline

이 디렉터리는 정적 웹페이지 페이지네이션 실습 `06_1 ~ 06_4`의  
**현재 코드 구조와 주요 함수의 역할**을 단계별로 설명합니다.

각 단계는 하나의 수집 배치를 기준으로 연결되며,  
`Crawling → Parsing → Preprocessing → MySQL Load` 순서로 실행됩니다.


## 전체 처리 흐름

```text
06_1 Crawling
run_crawling()
→ 웹페이지 요청
→ raw HTML 배치 폴더 생성
→ 페이지별 HTML 저장

        ↓

06_2 Parsing
run_extract(raw_batch_dir)
→ raw HTML 배치 읽기
→ 페이지별 HTML 파싱
→ interim CSV 배치 생성

        ↓

06_3 Preprocessing
run_preprocess(interim_batch_dir)
→ 페이지별 CSV 통합
→ 전처리 및 검증
→ processed CSV 생성

        ↓

06_4 MySQL Load
run_load(processed_csv_file)
→ processed 배치 검증
→ DB 저장용 자료형 변환
→ MySQL UPSERT
```


## 데이터 흐름

```text
data/raw/html/YYYYMMDD_HHMMSS/
            ↓
data/interim/YYYYMMDD_HHMMSS/
            ↓
data/processed/
└─ books_pages_001_003_processed_YYYYMMDD_HHMMSS.csv
            ↓
MySQL books
```

`raw`와 `interim`은 동일한 배치 이름을 사용합니다.

`processed`는 파일명에 페이지 범위와 동일한 배치 시각을 포함하여  
수집부터 DB 저장까지의 데이터 흐름을 추적할 수 있도록 구성합니다.


## 단계별 문서

| 단계 | 문서 | 내용 |
|---|---|---|
| 06_1 | [06_1_crawling.md](./06_1_crawling.md) | 페이지네이션 수집, 함수 리팩토링, raw 배치 폴더 생성 |
| 06_2 | [06_2_parsing.md](./06_2_parsing.md) | raw HTML 파싱 및 interim 배치 CSV 생성 |
| 06_3 | [06_3_preprocessing.md](./06_3_preprocessing.md) | CSV 통합, 전처리, 검증 및 processed CSV 생성 |
| 06_4 | [06_4_mysql_load.md](./06_4_mysql_load.md) | 배치 검증, DB 자료형 변환 및 MySQL UPSERT |


## 핵심 실행 함수

```text
06_1 Crawling
→ run_crawling()

06_2 Parsing
→ run_extract()

06_3 Preprocessing
→ run_preprocess()

06_4 MySQL Load
→ run_load()
```

각 단계의 핵심 실행 함수가 내부의 세부 함수들을 호출하여  
해당 단계의 전체 처리 흐름을 조정합니다.


## 단계별 입력과 반환값

| 단계 | 실행 함수 | 주요 입력 | 반환값 |
|---|---|---|---|
| 06_1 | `run_crawling()` | 페이지 범위 | raw HTML 배치 폴더 `Path` |
| 06_2 | `run_extract()` | raw HTML 배치 폴더 `Path` | parsed CSV `list[Path]` |
| 06_3 | `run_preprocess()` | interim 배치 폴더 `Path` | processed CSV `Path` |
| 06_4 | `run_load()` | processed CSV `Path` | MySQL 저장 결과 `dict` |


## 단계 연결 관계

현재 파이프라인은 **개별 파일이 아니라 배치 단위**로 연결됩니다.

```text
run_crawling()
       ↓
raw_batch_dir
       ↓
run_extract(raw_batch_dir)
       ↓
parsed_csv_files
       ↓
parsed_csv_files[0].parent
       ↓
interim_batch_dir
       ↓
run_preprocess(interim_batch_dir)
       ↓
processed_csv_file
       ↓
run_load(processed_csv_file)
       ↓
load_summary
```

`run_extract()`는 페이지별 parsed CSV 경로 목록을 반환합니다.

모든 parsed CSV는 같은 interim 배치 폴더에 저장되므로  
첫 번째 파일의 부모 경로를 이용해 다음 단계의 입력 배치를 구할 수 있습니다.

```python
interim_batch_dir = parsed_csv_files[0].parent
```


## 06_1 함수 구조

06_1은 기존의 절차형 페이지네이션 코드를  
함수 단위로 리팩토링한 구조입니다.

```text
run_crawling()
│
├─ create_batch_directory()
│  └─ ensure_directory()
│
└─ 페이지별 반복
   ├─ fetch_html()
   └─ save_raw_html()
```

주요 함수:

```text
fetch_html()
→ 웹페이지 요청 및 HTTP 응답 검증

ensure_directory()
→ 저장 폴더 생성

create_batch_directory()
→ 수집 시각을 이름으로 하는 raw 배치 폴더 생성

save_raw_html()
→ 페이지별 원본 HTML 저장

run_crawling()
→ 전체 페이지네이션 수집 흐름 조정
```


## 함수 수

| 단계 | 함수 수 | 핵심 실행 함수 |
|---|---:|---|
| 06_1 | 5 | `run_crawling()` |
| 06_2 | 14 | `run_extract()` |
| 06_3 | 18 | `run_preprocess()` |
| 06_4 | 12 | `run_load()` |
| **합계** | **49** | - |


## 처리 결과 예

1~3페이지를 한 번의 배치로 수집한 경우:

```text
06_1 Crawling

data/raw/html/20260810_081501/
├─ books_page_001.html
├─ books_page_002.html
└─ books_page_003.html

        ↓

06_2 Parsing

data/interim/20260810_081501/
├─ books_page_001_parsed.csv
├─ books_page_002_parsed.csv
└─ books_page_003_parsed.csv

        ↓

06_3 Preprocessing

data/processed/
└─ books_pages_001_003_processed_20260810_081501.csv

        ↓

06_4 MySQL Load

MySQL books
```


## 문서 기준

현재 문서는 각 단계의 최신 코드 구조를 기준으로 관리합니다.

```text
06_1
→ 페이지네이션 수집 코드를 함수 단위로 리팩토링한 구조

06_2
→ raw HTML 배치 기반 파싱 구조

06_3
→ interim 배치 기반 통합 및 전처리 구조

06_4
→ processed CSV 배치 검증 및 MySQL 저장 구조
```

특정 Step 번호를 전체 문서에 공통으로 적용하지 않고,  
각 단계의 현재 구현 구조를 기준으로 설명합니다.
