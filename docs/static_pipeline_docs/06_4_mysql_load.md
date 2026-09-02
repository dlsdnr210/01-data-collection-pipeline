# 06_4 MySQL Load
기준 Notebook:

```text
06_4_static_crawling_pagination_mysql_load_step02_batch_directorye.ipynb
```


## 단계 목적

processed CSV의 파일명과 내부 배치 정보를 검증하고,  
MySQL 저장용 자료형으로 변환한 뒤 SQLAlchemy를 이용해  
`books` 테이블에 UPSERT합니다.

```text
processed CSV 탐색
→ 파일명 배치 정보 추출
→ CSV 읽기
→ 배치 검증
→ DB 저장용 자료형 정리
→ Python 레코드 변환
→ MySQL 연결
→ books 테이블 생성
→ UPSERT
```


## 1. 전체 함수 흐름

```text
run_load()
│
├─ find_latest_processed_csv()
│  └─ parse_processed_file_name()
│
├─ load_processed_csv()
├─ validate_processed_batch()
├─ prepare_and_validate_dataframe()
├─ dataframe_to_database_records()
│
├─ load_database_config()
├─ create_mysql_engine()
│
├─ test_mysql_connection()
├─ create_books_table()
└─ upsert_books()
```

## 2. 함수 목록

| 순번 | 함수 | 반환 자료형 |
|---:|---|---|
| 1 | `parse_processed_file_name()` | `tuple[int, int, datetime]` |
| 2 | `find_latest_processed_csv()` | `Path` |
| 3 | `load_processed_csv()` | `pd.DataFrame` |
| 4 | `validate_processed_batch()` | `None` |
| 5 | `prepare_and_validate_dataframe()` | `pd.DataFrame` |
| 6 | `dataframe_to_database_records()` | `list[dict[str, Any]]` |
| 7 | `load_database_config()` | `dict[str, str | int]` |
| 8 | `create_mysql_engine()` | `Engine` |
| 9 | `test_mysql_connection()` | `dict[str, str]` |
| 10 | `create_books_table()` | `None` |
| 11 | `upsert_books()` | `int` |
| 12 | `run_load()` | `dict[str, int | str]` |

---

## 3. `parse_processed_file_name()`

```python
def parse_processed_file_name(file_path: Path) -> tuple[int, int, datetime]
```

### 역할

processed CSV 파일명에서 페이지 범위와 배치 시각을 추출한다.

### 매개변수

- `file_path` : `Path`

### 반환 자료형

```text
tuple[int, int, datetime]
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
processed CSV 파일명에서 페이지 범위와 배치 시각을 추출한다.

Args:
    file_path:
        페이지 범위와 배치 시각이 포함된 processed CSV 파일 경로

Returns:
    시작 페이지, 종료 페이지, 배치 시각의 튜플

Raises:
    ValueError:
        파일명이 지정한 규칙과 일치하지 않거나
        시작 페이지가 종료 페이지보다 큰 경우
```

## 4. `find_latest_processed_csv()`

```python
def find_latest_processed_csv(directory: Path=PROCESSED_DIR, pattern: str=PROCESSED_CSV_PATTERN) -> Path
```

### 역할

가장 최근 배치의 processed CSV 파일 경로를 반환한다.

### 매개변수

- `directory` : `Path`
- `pattern` : `str`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- `parse_processed_file_name()`

### 코드 Docstring

```text
가장 최근 배치의 processed CSV 파일 경로를 반환한다.

Args:
    directory:
        processed CSV 파일이 저장된 폴더

    pattern:
        검색할 processed CSV 파일명 패턴

Returns:
    최신 배치의 processed CSV 파일 경로

Raises:
    FileNotFoundError:
        폴더가 없거나 사용할 수 있는 processed CSV 파일이 없는 경우
```

## 5. `load_processed_csv()`

```python
def load_processed_csv(file_path: Path) -> pd.DataFrame
```

### 역할

processed CSV를 DataFrame으로 읽어 반환한다.

### 매개변수

- `file_path` : `Path`

### 반환 자료형

```text
pd.DataFrame
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
processed CSV를 DataFrame으로 읽어 반환한다.

Args:
    file_path:
        읽을 processed CSV 파일 경로

Returns:
    전처리 데이터가 저장된 DataFrame

Raises:
    FileNotFoundError:
        지정한 CSV 파일이 존재하지 않는 경우
```

## 6. `validate_processed_batch()`

```python
def validate_processed_batch(df: pd.DataFrame, start_page: int, end_page: int, batch_at: datetime) -> None
```

### 역할

processed 파일명의 배치 정보와 CSV 내부 데이터를 비교하여 검증한다.

### 매개변수

- `df` : `pd.DataFrame`
- `start_page` : `int`
- `end_page` : `int`
- `batch_at` : `datetime`

### 반환 자료형

```text
None
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
processed 파일명의 배치 정보와 CSV 내부 데이터를 비교하여 검증한다.

Args:
    df:
        processed CSV를 읽은 DataFrame

    start_page:
        processed 파일명에서 추출한 시작 페이지

    end_page:
        processed 파일명에서 추출한 종료 페이지

    batch_at:
        processed 파일명에서 추출한 배치 시각

Raises:
    ValueError:
        source_page 범위 또는 parsed_at이 파일명의 배치 정보와 다른 경우
```

## 7. `prepare_and_validate_dataframe()`

```python
def prepare_and_validate_dataframe(df: pd.DataFrame) -> pd.DataFrame
```

### 역할

DataFrame을 MySQL 저장용 자료형으로 정리하고 검증한다.

### 매개변수

- `df` : `pd.DataFrame`

### 반환 자료형

```text
pd.DataFrame
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
DataFrame을 MySQL 저장용 자료형으로 정리하고 검증한다.

Args:
    df:
        전처리가 완료된 도서 DataFrame

Returns:
    DB 저장용 컬럼과 자료형으로 정리된 DataFrame

Raises:
    ValueError:
        필수 컬럼, 결측값, 중복값 또는 값의 범위에 문제가 있는 경우
```

## 8. `dataframe_to_database_records()`

```python
def dataframe_to_database_records(df: pd.DataFrame) -> list[dict[str, Any]]
```

### 역할

DB 저장용 DataFrame을 Python 기본 자료형의 레코드 목록으로 변환한다.

### 매개변수

- `df` : `pd.DataFrame`

### 반환 자료형

```text
list[dict[str, Any]]
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
DB 저장용 DataFrame을 Python 기본 자료형의 레코드 목록으로 변환한다.

가격은 Decimal, 날짜는 datetime, 정수와 논리값은 Python 기본형으로 변환한다.

Args:
    df:
        검증이 완료된 DB 저장용 DataFrame

Returns:
    SQLAlchemy execute()에 전달할 레코드 딕셔너리 목록
```

## 9. `load_database_config()`

```python
def load_database_config(env_file: Path=ENV_FILE) -> dict[str, str | int]
```

### 역할

.env 파일에서 MySQL 연결 정보를 읽고 검증한다.

### 매개변수

- `env_file` : `Path`

### 반환 자료형

```text
dict[str, str | int]
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
.env 파일에서 MySQL 연결 정보를 읽고 검증한다.

Args:
    env_file:
        MySQL 연결 정보가 저장된 .env 파일 경로

Returns:
    host, port, database, username, password를 담은 연결 설정

Raises:
    FileNotFoundError:
        .env 파일이 존재하지 않는 경우

    ValueError:
        필수 환경 변수가 없거나 DB_PORT가 정수가 아닌 경우
```

## 10. `create_mysql_engine()`

```python
def create_mysql_engine(config: dict[str, str | int]) -> Engine
```

### 역할

MySQL 연결 설정으로 SQLAlchemy Engine을 생성한다.

### 매개변수

- `config` : `dict[str, str | int]`

### 반환 자료형

```text
Engine
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
MySQL 연결 설정으로 SQLAlchemy Engine을 생성한다.

Args:
    config:
        load_database_config()가 반환한 연결 설정

Returns:
    PyMySQL 드라이버를 사용하는 SQLAlchemy Engine
```

## 11. `test_mysql_connection()`

```python
def test_mysql_connection(engine: Engine) -> dict[str, str]
```

### 역할

MySQL 연결 상태와 서버 정보를 확인한다.

### 매개변수

- `engine` : `Engine`

### 반환 자료형

```text
dict[str, str]
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
MySQL 연결 상태와 서버 정보를 확인한다.

Args:
    engine:
        연결을 확인할 SQLAlchemy Engine

Returns:
    MySQL 버전, 데이터베이스명, 현재 사용자 정보
```

## 12. `create_books_table()`

```python
def create_books_table(engine: Engine) -> None
```

### 역할

books 테이블이 없으면 생성한다.

### 매개변수

- `engine` : `Engine`

### 반환 자료형

```text
None
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
books 테이블이 없으면 생성한다.
```

## 13. `upsert_books()`

```python
def upsert_books(engine: Engine, records: list[dict[str, Any]]) -> int
```

### 역할

도서 레코드를 books 테이블에 UPSERT한다.

### 매개변수

- `engine` : `Engine`
- `records` : `list[dict[str, Any]]`

### 반환 자료형

```text
int
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
도서 레코드를 books 테이블에 UPSERT한다.

새로운 book_id는 INSERT하고 기존 book_id는 UPDATE한다.

Args:
    engine:
        MySQL SQLAlchemy Engine

    records:
        저장할 도서 레코드 목록

Returns:
    DB 드라이버가 보고한 영향 행 수
```

## 14. `run_load()`

```python
def run_load(processed_csv_file: Path | None=None, engine: Engine | None=None) -> dict[str, int | str]
```

### 역할

processed CSV의 배치를 검증한 뒤 MySQL 저장까지 순서대로 실행한다.

### 매개변수

- `processed_csv_file` : `Path | None`
- `engine` : `Engine | None`

### 반환 자료형

```text
dict[str, int | str]
```

### 내부에서 호출하는 현재 단계 함수

- `parse_processed_file_name()`
- `load_processed_csv()`
- `validate_processed_batch()`
- `prepare_and_validate_dataframe()`
- `dataframe_to_database_records()`
- `find_latest_processed_csv()`
- `load_database_config()`
- `create_mysql_engine()`
- `test_mysql_connection()`
- `create_books_table()`
- `upsert_books()`

### 코드 Docstring

```text
processed CSV의 배치를 검증한 뒤 MySQL 저장까지 순서대로 실행한다.

Args:
    processed_csv_file:
        MySQL에 저장할 processed CSV 파일 경로

        값을 전달하지 않으면 data/processed 폴더에서
        가장 최근 배치의 파일을 자동으로 찾는다.

    engine:
        외부에서 생성한 SQLAlchemy Engine

        값을 전달하지 않으면 .env 설정으로 새 Engine을 생성한다.

Returns:
    입력 파일명, 배치 시각, 데이터베이스명, 입력 행 수,
    DB 드라이버 영향 행 수가 포함된 요약 정보
```

## MySQL 저장 흐름

```text
data/processed/
└─ books_pages_001_003_processed_20260809_233901.csv
            ↓
배치 검증
            ↓
DB 저장용 변환
            ↓
SQLAlchemy Engine
            ↓
MySQL books
```

## 핵심 실행 함수

```python
run_load()
```

06_4 단계의 파일 탐색, 배치 검증, DB 변환, 연결, 테이블 생성, UPSERT를 조정합니다.

## 주요 시간 컬럼

```text
created_at
→ 최초 INSERT 시각

updated_at
→ 주요 도서 데이터가 실제로 변경된 시각

last_checked_at
→ 해당 도서를 최근 수집에서 다시 확인한 시각
```

`affected_row_count`는 테이블 전체 행 수가 아니라  
DB 드라이버가 UPSERT 결과로 보고한 영향 행 수입니다.
