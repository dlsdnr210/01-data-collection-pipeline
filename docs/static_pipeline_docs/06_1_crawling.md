# 06_1 Crawling

기준 Notebook:

```text
06_1_static_crawling_pagination_step03_refactoring.ipynb
```


## 단계 목적

Books to Scrape의 여러 페이지를 순차적으로 요청하고,  
같은 실행에서 수집한 원본 HTML을 하나의 **raw 배치 디렉터리**에 저장합니다.

기존 절차형 페이지네이션 코드를 함수 단위로 리팩토링하여  
Python 모듈의 `run_crawling()`을 `main.py`에서 호출할 수 있도록 구성합니다.

```text
페이지 URL 생성
→ 웹페이지 요청
→ HTTP 응답 검증
→ 배치 디렉터리 생성
→ 페이지별 HTML 저장
→ raw 배치 디렉터리 반환
```


## 1. 전체 함수 흐름

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

`run_crawling()`은 페이지별 HTML 파일 목록이 아니라  
**이번 수집 작업의 raw HTML 배치 디렉터리 `Path`**를 반환합니다.

```text
run_crawling()
       ↓
data/raw/html/YYYYMMDD_HHMMSS/
       ↓
run_extract()
```


## 2. 함수 목록

| 순번 | 함수 | 역할 | 반환 자료형 |
|---:|---|---|---|
| 1 | `fetch_html()` | 지정한 URL에 GET 요청을 보내고 응답 검증 | `requests.Response` |
| 2 | `ensure_directory()` | 저장 폴더 생성 및 경로 반환 | `Path` |
| 3 | `create_batch_directory()` | 수집 시각을 이름으로 하는 배치 폴더 생성 | `Path` |
| 4 | `save_raw_html()` | 페이지별 원본 HTML 저장 | `Path` |
| 5 | `run_crawling()` | 전체 페이지네이션 수집 작업 실행 | `Path` |


---

## 3. 주요 설정

```python
BASE_URL = 'https://books.toscrape.com/catalogue/'

START_PAGE = 1
END_PAGE = 3

CONNECT_TIMEOUT = 5
READ_TIMEOUT = 30

REQUEST_INTERVAL = 0.5

HEADERS = {'User-Agent': 'EducationalDataCollector/1.0'}
```

### 역할

- `BASE_URL`: 페이지 URL의 공통 부분
- `START_PAGE`: 수집 시작 페이지
- `END_PAGE`: 수집 종료 페이지
- `CONNECT_TIMEOUT`: 서버 연결 제한 시간
- `READ_TIMEOUT`: 응답 데이터 대기 제한 시간
- `REQUEST_INTERVAL`: 연속 요청 사이의 대기 시간(초)
- `HEADERS`: HTTP 요청에 사용할 헤더


## 4. 프로젝트 및 저장 경로

Notebook에서는 실행 위치를 기준으로 프로젝트 루트를 계산합니다.

```python
PROJECT_DIR = Path.cwd().resolve().parents[1]
RAW_HTML_DIR = PROJECT_DIR / 'data' / 'raw' / 'html'
```

Python 모듈인 `crawling.py`로 변환하여 사용할 때는  
`__file__`을 기준으로 프로젝트 루트를 계산하도록 구성할 수 있습니다.

최종 원본 HTML 저장 구조는 동일합니다.

```text
data/raw/html/
└─ YYYYMMDD_HHMMSS/
   ├─ books_page_001.html
   ├─ books_page_002.html
   └─ ...
```


## 5. `fetch_html()`

```python
def fetch_html(
    url: str,
    connect_timeout: int = CONNECT_TIMEOUT,
    read_timeout: int = READ_TIMEOUT,
) -> requests.Response
```

### 역할

지정한 URL에 HTTP GET 요청을 보내고 정상 응답인지 확인한 뒤  
`requests.Response` 객체를 반환합니다.

### 매개변수

- `url` : `str`
  - 요청할 웹페이지 URL
- `connect_timeout` : `int`
  - 서버 연결 제한 시간
- `read_timeout` : `int`
  - 응답 데이터 대기 제한 시간

### 반환 자료형

```text
requests.Response
```

### 주요 처리

```python
response = requests.get(
    url,
    headers=HEADERS,
    timeout=(connect_timeout, read_timeout),
)

response.raise_for_status()
```

`url`, `connect_timeout`, `read_timeout`은 함수에 전달된 값을  
실제 HTTP 요청에 그대로 사용합니다.

### 예외

`response.raise_for_status()`를 포함한 요청 과정에서 문제가 발생하면  
`requests.exceptions.RequestException` 계열 예외가 발생할 수 있습니다.

### 내부에서 호출하는 현재 단계 함수

- 없음


## 6. `ensure_directory()`

```python
def ensure_directory(directory: Path) -> Path
```

### 역할

지정한 폴더가 없으면 생성하고 해당 폴더의 `Path`를 반환합니다.

### 매개변수

- `directory` : `Path`
  - 생성하거나 확인할 폴더 경로

### 반환 자료형

```text
Path
```

### 주요 처리

```python
directory.mkdir(
    parents=True,
    exist_ok=True,
)
```

- `parents=True`
  - 상위 폴더가 없으면 함께 생성
- `exist_ok=True`
  - 폴더가 이미 있어도 오류를 발생시키지 않음

### 내부에서 호출하는 현재 단계 함수

- 없음


## 7. `create_batch_directory()`

```python
def create_batch_directory(
    directory: Path,
    collected_at: datetime,
) -> Path
```

### 역할

전체 수집 시작 시각을 `YYYYMMDD_HHMMSS` 형식으로 변환하여  
한 번의 크롤링 실행을 구분하는 raw HTML 배치 폴더를 생성합니다.

### 매개변수

- `directory` : `Path`
  - 모든 수집 배치가 저장되는 기본 폴더
- `collected_at` : `datetime`
  - 전체 수집 작업의 시작 시각

### 반환 자료형

```text
Path
```

### 주요 처리

```python
batch_name = collected_at.strftime('%Y%m%d_%H%M%S')
batch_dir = directory / batch_name

return ensure_directory(batch_dir)
```

예:

```text
2026-08-10 08:15:01
        ↓
20260810_081501
        ↓
data/raw/html/20260810_081501/
```

### 내부에서 호출하는 현재 단계 함수

- `ensure_directory()`


## 8. `save_raw_html()`

```python
def save_raw_html(
    content: bytes,
    batch_dir: Path,
    source_page: int,
) -> Path
```

### 역할

서버에서 받은 페이지별 원본 HTML 바이트 데이터를  
현재 수집 배치 폴더에 저장합니다.

### 매개변수

- `content` : `bytes`
  - 서버에서 받은 원본 응답 본문
- `batch_dir` : `Path`
  - 원본 HTML을 저장할 raw 배치 폴더
- `source_page` : `int`
  - 수집한 페이지 번호

### 반환 자료형

```text
Path
```

### 파일명

```python
file_path = batch_dir / f'books_page_{source_page:03d}.html'
```

예:

```text
books_page_001.html
books_page_002.html
books_page_003.html
```

배치 시각은 상위 디렉터리 이름이 관리하므로  
페이지별 HTML 파일명에는 timestamp를 반복하지 않습니다.

### 저장

```python
file_path.write_bytes(content)
```

### 내부에서 호출하는 현재 단계 함수

- 없음


## 9. `run_crawling()`

```python
def run_crawling(
    base_url: str = BASE_URL,
    start_page: int = START_PAGE,
    end_page: int = END_PAGE,
) -> Path
```

### 역할

페이지 범위를 순회하면서 웹페이지를 요청하고,  
페이지별 원본 HTML을 하나의 raw 배치 폴더에 저장하는  
**06_1 단계의 핵심 실행 함수**입니다.

### 매개변수

- `base_url` : `str`
  - 페이지 URL의 공통 부분
- `start_page` : `int`
  - 수집 시작 페이지
- `end_page` : `int`
  - 수집 종료 페이지

### 반환 자료형

```text
Path
```

반환되는 `Path`는 개별 HTML 파일이 아니라  
이번 수집 작업의 **raw HTML 배치 디렉터리**입니다.

예:

```text
data/raw/html/20260810_081501/
```

### 페이지 범위 검증

```python
if start_page <= 0 or end_page <= 0:
    raise ValueError('페이지 번호는 1 이상이어야 합니다.')

if start_page > end_page:
    raise ValueError('시작 페이지는 종료 페이지보다 클 수 없습니다.')
```

### 내부 처리 흐름

```text
datetime.now()
→ 수집 시작 시각 생성
→ create_batch_directory()
→ raw 배치 폴더 생성

→ start_page ~ end_page 반복
   ├─ 페이지 URL 생성
   ├─ fetch_html()
   ├─ save_raw_html()
   ├─ 저장된 Path를 raw_files에 추가
   └─ 마지막 페이지가 아니면 일정 시간 대기

→ 수집 결과 출력
→ batch_dir 반환
```

### 내부에서 호출하는 현재 단계 함수

- `create_batch_directory()`
- `fetch_html()`
- `save_raw_html()`


## 10. 페이지네이션 처리

페이지 번호를 이용해 요청 URL을 생성합니다.

```python
target_url = f'{base_url}page-{page}.html'
```

예:

```text
1페이지
→ https://books.toscrape.com/catalogue/page-1.html

2페이지
→ https://books.toscrape.com/catalogue/page-2.html

3페이지
→ https://books.toscrape.com/catalogue/page-3.html
```

각 페이지의 처리 흐름은 다음과 같습니다.

```text
페이지 URL 생성
→ fetch_html()
→ HTTP 응답 검증
→ save_raw_html()
→ raw_files에 저장 경로 추가
```


## 11. 요청 간 대기

마지막 페이지가 아니라면 다음 요청 전에 대기합니다.

```python
if page < end_page:
    time.sleep(REQUEST_INTERVAL)
```

웹 서버에 지나치게 빠르게 연속 요청하지 않도록 하기 위한 처리입니다.


## 12. 저장 결과

1~3페이지를 수집했다면 다음과 같이 저장됩니다.

```text
data/raw/html/
└─ 20260810_081501/
   ├─ books_page_001.html
   ├─ books_page_002.html
   └─ books_page_003.html
```

하나의 배치 폴더에는 같은 `run_crawling()` 실행에서 수집한  
페이지별 원본 HTML만 저장됩니다.


## 13. 모듈 단독 실행

`crawling.py`를 직접 실행할 때는 다음 구조를 사용합니다.

```python
if __name__ == '__main__':
    try:
        run_crawling()

    except requests.exceptions.RequestException as error:
        ...

    except (OSError, ValueError) as error:
        ...
```

다른 모듈에서 `crawling.py`를 import하는 경우에는  
이 블록이 자동 실행되지 않습니다.

따라서 `main.py`에서 `run_crawling()`만 가져와 호출할 수 있습니다.


## 14. `main.py`와 연결

현재 전체 파이프라인에서 06_1은 다음과 같이 연결됩니다.

```python
raw_batch_dir = run_crawling()
parsed_csv_files = run_extract(raw_batch_dir)
```

데이터 흐름:

```text
run_crawling()
       ↓
raw_batch_dir
       ↓
data/raw/html/YYYYMMDD_HHMMSS/
       ↓
run_extract(raw_batch_dir)
```

즉, `run_crawling()`의 반환값과 `run_extract()`의 입력값은  
동일한 **raw HTML 배치 디렉터리 `Path`**입니다.


## 15. 리팩토링 전후 비교

### 리팩토링 전

페이지 요청, 폴더 생성, HTML 저장이 하나의 실행 블록에 함께 작성되어 있었습니다.

```text
설정
→ 배치 폴더 생성
→ for 반복
→ requests.get()
→ HTML 저장
→ 결과 출력
```

### 리팩토링 후

각 기능을 독립된 함수로 분리했습니다.

```text
fetch_html()
→ HTTP 요청

ensure_directory()
→ 폴더 생성

create_batch_directory()
→ 배치 폴더 생성

save_raw_html()
→ 원본 HTML 저장

run_crawling()
→ 전체 페이지네이션 수집 흐름 조정
```

`run_crawling()`은 세부 구현을 직접 모두 담당하기보다  
각 역할을 담당하는 함수를 호출하여 전체 수집 흐름을 조정합니다.


## 핵심 실행 함수

```python
run_crawling()
```

06_1 단계의 웹 요청, 배치 폴더 생성, 페이지별 HTML 저장을 연결하고  
최종적으로 raw HTML 배치 폴더의 `Path`를 반환합니다.
