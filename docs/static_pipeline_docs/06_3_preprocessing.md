# 06_3 Preprocessing
기준 Notebook:

```text
06_3_static_crawling_pagination_preprocessing_step02_batch_directory.ipynb
```


## 단계 목적

최신 interim 배치의 페이지별 파싱 CSV를 하나의 DataFrame으로 통합하고,  
자료형 변환·중복 처리·메타데이터 생성·데이터 검증을 수행한 뒤  
`data/processed`에 하나의 CSV로 저장합니다.

```text
interim 배치 탐색
→ 페이지별 CSV 정렬 및 검증
→ CSV 통합
→ 값 전처리
→ 중복 제거
→ 최종 검증
→ processed CSV 저장
```


## 1. 전체 함수 흐름

```text
run_preprocess()
│
├─ find_latest_interim_batch_directory()
│  └─ parse_batch_directory_name()
│
├─ find_parsed_csv_files()
│  └─ parse_parsed_file_name()
│
├─ load_parsed_csv_files()
│  ├─ load_parsed_csv()
│  └─ validate_input_books()
│
├─ preprocess_books()
│  ├─ clean_string_columns()
│  ├─ parse_price()
│  └─ parse_availability()
│
├─ validate_processed_books()
│
├─ save_processed_csv()
│  ├─ build_processed_file_path()
│  └─ save_csv_atomically()
│     └─ ensure_directory()
│
└─ verify_saved_csv()
```

## 2. 함수 목록

| 순번 | 함수 | 반환 자료형 |
|---:|---|---|
| 1 | `parse_batch_directory_name()` | `datetime` |
| 2 | `find_latest_interim_batch_directory()` | `Path` |
| 3 | `parse_parsed_file_name()` | `int` |
| 4 | `find_parsed_csv_files()` | `tuple[list[Path], list[int]]` |
| 5 | `load_parsed_csv()` | `pd.DataFrame` |
| 6 | `validate_input_books()` | `None` |
| 7 | `load_parsed_csv_files()` | `pd.DataFrame` |
| 8 | `clean_string_columns()` | `pd.DataFrame` |
| 9 | `parse_price()` | `pd.Series` |
| 10 | `parse_availability()` | `bool | None` |
| 11 | `preprocess_books()` | `pd.DataFrame` |
| 12 | `validate_processed_books()` | `dict[str, int]` |
| 13 | `ensure_directory()` | `Path` |
| 14 | `save_csv_atomically()` | `Path` |
| 15 | `build_processed_file_path()` | `Path` |
| 16 | `save_processed_csv()` | `Path` |
| 17 | `verify_saved_csv()` | `pd.DataFrame` |
| 18 | `run_preprocess()` | `Path` |

---

## 3. `parse_batch_directory_name()`

```python
def parse_batch_directory_name(batch_dir: Path) -> datetime
```

### 역할

interim 배치 폴더명에서 배치 시각을 추출한다.

### 매개변수

- `batch_dir` : `Path`

### 반환 자료형

```text
datetime
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
interim 배치 폴더명에서 배치 시각을 추출한다.

Args:
    batch_dir:
        YYYYMMDD_HHMMSS 형식의 interim 배치 폴더 경로

Returns:
    배치 폴더명에서 추출한 날짜와 시각

Raises:
    ValueError:
        폴더명이 지정한 형식과 일치하지 않는 경우

Examples:
    data/interim/20260809_233901
```

## 4. `find_latest_interim_batch_directory()`

```python
def find_latest_interim_batch_directory(directory: Path=INTERIM_DIR) -> Path
```

### 역할

data/interim 폴더에서 가장 최근 배치 폴더를 반환한다.

### 매개변수

- `directory` : `Path`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- `parse_batch_directory_name()`

### 코드 Docstring

```text
data/interim 폴더에서 가장 최근 배치 폴더를 반환한다.

Args:
    directory:
        interim 배치 폴더들이 저장된 기본 경로

Returns:
    가장 최근 interim 배치 폴더 경로

Raises:
    FileNotFoundError:
        interim 폴더가 없거나 유효한 배치 폴더가 없는 경우
```

## 5. `parse_parsed_file_name()`

```python
def parse_parsed_file_name(file_path: Path) -> int
```

### 역할

파싱 CSV 파일명에서 페이지 번호를 추출한다.

### 매개변수

- `file_path` : `Path`

### 반환 자료형

```text
int
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
파싱 CSV 파일명에서 페이지 번호를 추출한다.

Args:
    file_path:
        books_page_NNN_parsed.csv 형식의 파싱 CSV 파일 경로

Returns:
    파싱 CSV의 페이지 번호

Raises:
    ValueError:
        파일명이 지정한 규칙과 일치하지 않는 경우

Examples:
    books_page_001_parsed.csv
```

## 6. `find_parsed_csv_files()`

```python
def find_parsed_csv_files(batch_dir: Path) -> tuple[list[Path], list[int]]
```

### 역할

interim 배치 폴더의 파싱 CSV를 페이지 순서대로 반환한다.

### 매개변수

- `batch_dir` : `Path`

### 반환 자료형

```text
tuple[list[Path], list[int]]
```

### 내부에서 호출하는 현재 단계 함수

- `parse_parsed_file_name()`

### 코드 Docstring

```text
interim 배치 폴더의 파싱 CSV를 페이지 순서대로 반환한다.

페이지 번호가 중복되거나 중간 페이지가 누락된 경우 오류를 발생시킨다.

Args:
    batch_dir:
        페이지별 파싱 CSV가 저장된 interim 배치 폴더

Returns:
    페이지 순으로 정렬된 CSV 파일 목록과 페이지 번호 목록

Raises:
    FileNotFoundError:
        배치 폴더가 없거나 파싱 CSV 파일이 없는 경우

    ValueError:
        페이지 번호가 중복되거나 누락된 경우
```

## 7. `load_parsed_csv()`

```python
def load_parsed_csv(file_path: Path) -> pd.DataFrame
```

### 역할

파싱 CSV 파일 한 개를 DataFrame으로 읽어 반환한다.

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
파싱 CSV 파일 한 개를 DataFrame으로 읽어 반환한다.

Args:
    file_path:
        읽을 파싱 CSV 파일 경로

Returns:
    파싱 데이터가 저장된 DataFrame

Raises:
    FileNotFoundError:
        지정한 CSV 파일이 존재하지 않는 경우
```

## 8. `validate_input_books()`

```python
def validate_input_books(books_df: pd.DataFrame) -> None
```

### 역할

전처리 입력 DataFrame의 행과 필수 컬럼을 검증한다.

### 매개변수

- `books_df` : `pd.DataFrame`

### 반환 자료형

```text
None
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
전처리 입력 DataFrame의 행과 필수 컬럼을 검증한다.

Args:
    books_df:
        검증할 파싱 DataFrame

Raises:
    ValueError:
        DataFrame이 비어 있거나 필수 컬럼이 없는 경우
```

## 9. `load_parsed_csv_files()`

```python
def load_parsed_csv_files(parsed_csv_files: list[Path]) -> pd.DataFrame
```

### 역할

페이지별 파싱 CSV를 읽어 하나의 DataFrame으로 통합한다.

### 매개변수

- `parsed_csv_files` : `list[Path]`

### 반환 자료형

```text
pd.DataFrame
```

### 내부에서 호출하는 현재 단계 함수

- `parse_parsed_file_name()`
- `load_parsed_csv()`
- `validate_input_books()`

### 코드 Docstring

```text
페이지별 파싱 CSV를 읽어 하나의 DataFrame으로 통합한다.

파일명의 페이지 번호와 CSV 내부 source_page 값도 함께 검증한다.

Args:
    parsed_csv_files:
        페이지 순서대로 정렬된 파싱 CSV 파일 경로 목록

Returns:
    모든 페이지가 통합된 DataFrame

Raises:
    ValueError:
        파일명 페이지 번호와 CSV 내부 페이지 번호가 다른 경우
```

## 10. `clean_string_columns()`

```python
def clean_string_columns(df: pd.DataFrame) -> pd.DataFrame
```

### 역할

문자열 컬럼을 Pandas string dtype으로 변환하고 앞뒤 공백을 제거한다.

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
문자열 컬럼을 Pandas string dtype으로 변환하고 앞뒤 공백을 제거한다.

빈 문자열은 pd.NA로 변환한다.

Args:
    df:
        문자열 컬럼을 정리할 DataFrame

Returns:
    문자열 컬럼이 정리된 새로운 DataFrame
```

## 11. `parse_price()`

```python
def parse_price(price_series: pd.Series) -> pd.Series
```

### 역할

가격 문자열에서 숫자를 추출하여 Float64 자료형으로 변환한다.

### 매개변수

- `price_series` : `pd.Series`

### 반환 자료형

```text
pd.Series
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
가격 문자열에서 숫자를 추출하여 Float64 자료형으로 변환한다.

Args:
    price_series:
        가격 문자열 Series

Returns:
    Float64 자료형의 가격 Series
```

## 12. `parse_availability()`

```python
def parse_availability(value: object) -> bool | None
```

### 역할

재고 상태 문자열을 논리값으로 변환한다.

### 매개변수

- `value` : `object`

### 반환 자료형

```text
bool | None
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
재고 상태 문자열을 논리값으로 변환한다.

Args:
    value:
        재고 상태 값

Returns:
    재고 있음은 True, 재고 없음은 False,
    판단할 수 없는 값은 None
```

## 13. `preprocess_books()`

```python
def preprocess_books(books_df: pd.DataFrame, batch_at: datetime) -> pd.DataFrame
```

### 역할

통합된 도서 데이터를 분석 가능한 구조로 전처리한다.

### 매개변수

- `books_df` : `pd.DataFrame`
- `batch_at` : `datetime`

### 반환 자료형

```text
pd.DataFrame
```

### 내부에서 호출하는 현재 단계 함수

- `validate_input_books()`
- `clean_string_columns()`
- `parse_price()`

### 코드 Docstring

```text
통합된 도서 데이터를 분석 가능한 구조로 전처리한다.

Args:
    books_df:
        모든 페이지가 통합된 파싱 DataFrame

    batch_at:
        interim 배치 폴더명에 포함된 배치 시각

Returns:
    전처리와 페이지 간 중복 제거가 완료된 DataFrame
```

## 14. `validate_processed_books()`

```python
def validate_processed_books(df: pd.DataFrame) -> dict[str, int]
```

### 역할

전처리된 도서 데이터의 필수 컬럼과 값의 품질을 검증한다.

### 매개변수

- `df` : `pd.DataFrame`

### 반환 자료형

```text
dict[str, int]
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
전처리된 도서 데이터의 필수 컬럼과 값의 품질을 검증한다.

Args:
    df:
        검증할 전처리 DataFrame

Returns:
    행 수, 컬럼 수, 중복 수, 결측값 수를 담은 검증 요약

Raises:
    ValueError:
        하나 이상의 검증 규칙을 통과하지 못한 경우
```

## 15. `ensure_directory()`

```python
def ensure_directory(directory: Path) -> Path
```

### 역할

지정한 폴더가 없으면 생성하고 폴더 경로를 반환한다.

### 매개변수

- `directory` : `Path`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
지정한 폴더가 없으면 생성하고 폴더 경로를 반환한다.
```

## 16. `save_csv_atomically()`

```python
def save_csv_atomically(df: pd.DataFrame, file_path: Path) -> Path
```

### 역할

DataFrame을 임시 CSV에 저장한 뒤 최종 파일로 교체한다.

### 매개변수

- `df` : `pd.DataFrame`
- `file_path` : `Path`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- `ensure_directory()`

### 코드 Docstring

```text
DataFrame을 임시 CSV에 저장한 뒤 최종 파일로 교체한다.

Args:
    df:
        저장할 DataFrame

    file_path:
        최종 CSV 파일 경로

Returns:
    저장된 최종 CSV 파일 경로
```

## 17. `build_processed_file_path()`

```python
def build_processed_file_path(source_pages: list[int], batch_at: datetime, directory: Path=PROCESSED_DIR) -> Path
```

### 역할

페이지 범위와 배치 시각으로 전처리 CSV 파일 경로를 생성한다.

### 매개변수

- `source_pages` : `list[int]`
- `batch_at` : `datetime`
- `directory` : `Path`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
페이지 범위와 배치 시각으로 전처리 CSV 파일 경로를 생성한다.

Args:
    source_pages:
        전처리한 페이지 번호 목록

    batch_at:
        interim 배치 폴더명에서 추출한 배치 시각

    directory:
        전처리 CSV 저장 폴더

Returns:
    생성된 전처리 CSV 파일 경로
```

## 18. `save_processed_csv()`

```python
def save_processed_csv(processed_df: pd.DataFrame, source_pages: list[int], batch_at: datetime) -> Path
```

### 역할

전처리 DataFrame을 하나의 processed CSV 파일로 저장한다.

### 매개변수

- `processed_df` : `pd.DataFrame`
- `source_pages` : `list[int]`
- `batch_at` : `datetime`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- `build_processed_file_path()`
- `save_csv_atomically()`

### 코드 Docstring

```text
전처리 DataFrame을 하나의 processed CSV 파일로 저장한다.
```

## 19. `verify_saved_csv()`

```python
def verify_saved_csv(saved_file: Path, original_df: pd.DataFrame) -> pd.DataFrame
```

### 역할

저장한 CSV를 다시 읽고 행 수와 컬럼 순서를 검증한다.

### 매개변수

- `saved_file` : `Path`
- `original_df` : `pd.DataFrame`

### 반환 자료형

```text
pd.DataFrame
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
저장한 CSV를 다시 읽고 행 수와 컬럼 순서를 검증한다.

Args:
    saved_file:
        저장한 CSV 파일 경로

    original_df:
        저장 전 전처리 DataFrame

Returns:
    다시 읽은 CSV DataFrame

Raises:
    ValueError:
        저장 전후의 행 수 또는 컬럼 순서가 다른 경우
```

## 20. `run_preprocess()`

```python
def run_preprocess(interim_batch_dir: Path | None=None) -> Path
```

### 역할

interim 배치의 CSV를 통합하여 전처리하고 하나의 CSV로 저장한다.

### 매개변수

- `interim_batch_dir` : `Path | None`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- `parse_batch_directory_name()`
- `find_parsed_csv_files()`
- `load_parsed_csv_files()`
- `preprocess_books()`
- `validate_processed_books()`
- `save_processed_csv()`
- `verify_saved_csv()`
- `find_latest_interim_batch_directory()`

### 코드 Docstring

```text
interim 배치의 CSV를 통합하여 전처리하고 하나의 CSV로 저장한다.

Args:
    interim_batch_dir:
        전처리할 interim 배치 폴더

        값을 전달하지 않으면 data/interim 폴더에서
        가장 최근 배치 폴더를 자동으로 찾는다.

Returns:
    저장된 전처리 CSV 파일 경로
```

## 처리 결과

예를 들어 1~3페이지 배치를 전처리하면:

```text
data/interim/20260809_233901/
            ↓
data/processed/
└─ books_pages_001_003_processed_20260809_233901.csv
```

## 핵심 실행 함수

```python
run_preprocess()
```

06_3 단계의 배치 탐색, CSV 통합, 전처리, 검증, 저장을 조정합니다.
