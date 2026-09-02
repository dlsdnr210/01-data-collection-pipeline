# 06_2 Parsing
기준 Notebook:

```text
06_2_static_crawling_pagination_parsing_step02_batch_directory.ipynb
```


## 단계 목적

06_1에서 생성한 raw HTML 배치를 찾아 페이지별 도서 정보를 파싱하고,  
raw와 같은 배치 이름으로 `data/interim`에 CSV를 저장합니다.

```text
raw 배치 탐색
→ HTML 파일 정렬 및 검증
→ HTML 읽기
→ 도서 정보 추출
→ 파싱 결과 검증
→ interim 배치 CSV 저장
```


## 1. 전체 함수 흐름

```text
run_extract()
│
├─ find_latest_batch_directory()
│  └─ parse_batch_directory_name()
│
├─ find_raw_html_files()
│  └─ parse_raw_file_name()
│
├─ create_interim_batch_directory()
│
└─ 페이지별 반복
   ├─ load_raw_html()
   ├─ parse_books()
   │  └─ parse_book_item()
   │     ├─ get_required_tag()
   │     └─ parse_rating()
   ├─ validate_books_dataframe()
   ├─ save_parsed_csv()
   └─ verify_saved_csv()
```

## 2. 함수 목록

| 순번 | 함수 | 반환 자료형 |
|---:|---|---|
| 1 | `parse_batch_directory_name()` | `datetime` |
| 2 | `find_latest_batch_directory()` | `Path` |
| 3 | `parse_raw_file_name()` | `int` |
| 4 | `find_raw_html_files()` | `list[Path]` |
| 5 | `load_raw_html()` | `bytes` |
| 6 | `get_required_tag()` | `Tag` |
| 7 | `parse_rating()` | `tuple[str, int]` |
| 8 | `parse_book_item()` | `dict[str, str | int]` |
| 9 | `parse_books()` | `pd.DataFrame` |
| 10 | `validate_books_dataframe()` | `int` |
| 11 | `create_interim_batch_directory()` | `Path` |
| 12 | `save_parsed_csv()` | `Path` |
| 13 | `verify_saved_csv()` | `pd.DataFrame` |
| 14 | `run_extract()` | `list[Path]` |

---

## 3. `parse_batch_directory_name()`

```python
def parse_batch_directory_name(batch_dir: Path) -> datetime
```

### 역할

수집 배치 폴더명에서 수집 시각을 추출한다.

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
수집 배치 폴더명에서 수집 시각을 추출한다.

Args:
    batch_dir:
        YYYYMMDD_HHMMSS 형식의 수집 배치 폴더 경로

Returns:
    배치 폴더명에서 추출한 원본 HTML 수집 시각

Raises:
    ValueError:
        폴더명이 지정한 형식과 일치하지 않는 경우

Examples:
    data/raw/html/20260809_224616
```

## 4. `find_latest_batch_directory()`

```python
def find_latest_batch_directory(directory: Path=RAW_HTML_DIR) -> Path
```

### 역할

data/raw/html 폴더에서 가장 최근 수집 배치 폴더를 반환한다.

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
data/raw/html 폴더에서 가장 최근 수집 배치 폴더를 반환한다.

Args:
    directory:
        수집 배치 폴더들이 저장된 기본 HTML 폴더

Returns:
    가장 최근 수집 배치 폴더 경로

Raises:
    FileNotFoundError:
        기본 HTML 폴더가 없거나 유효한 수집 배치 폴더가 없는 경우
```

## 5. `parse_raw_file_name()`

```python
def parse_raw_file_name(file_path: Path) -> int
```

### 역할

원본 HTML 파일명에서 페이지 번호를 추출한다.

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
원본 HTML 파일명에서 페이지 번호를 추출한다.

Args:
    file_path:
        books_page_NNN.html 형식의 원본 HTML 파일 경로

Returns:
    원본 HTML의 페이지 번호

Raises:
    ValueError:
        파일명이 지정한 규칙과 일치하지 않는 경우

Examples:
    books_page_001.html
```

## 6. `find_raw_html_files()`

```python
def find_raw_html_files(batch_dir: Path) -> list[Path]
```

### 역할

수집 배치 폴더의 HTML 파일을 페이지 순서대로 반환한다.

### 매개변수

- `batch_dir` : `Path`

### 반환 자료형

```text
list[Path]
```

### 내부에서 호출하는 현재 단계 함수

- `parse_raw_file_name()`

### 코드 Docstring

```text
수집 배치 폴더의 HTML 파일을 페이지 순서대로 반환한다.

페이지 번호가 중복되거나 중간 페이지가 누락된 경우 오류를 발생시킨다.

Args:
    batch_dir:
        페이지별 원본 HTML 파일이 저장된 수집 배치 폴더

Returns:
    페이지 번호순으로 정렬된 원본 HTML 파일 경로 목록

Raises:
    FileNotFoundError:
        배치 폴더가 없거나 파싱할 HTML 파일이 없는 경우

    ValueError:
        페이지 번호가 중복되거나 누락된 경우
```

## 7. `load_raw_html()`

```python
def load_raw_html(file_path: Path) -> bytes
```

### 역할

원본 HTML 파일을 바이트 데이터로 읽어 반환한다.

### 매개변수

- `file_path` : `Path`

### 반환 자료형

```text
bytes
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
원본 HTML 파일을 바이트 데이터로 읽어 반환한다.

Args:
    file_path:
        읽을 원본 HTML 파일 경로

Returns:
    원본 HTML 바이트 데이터

Raises:
    FileNotFoundError:
        지정한 원본 HTML 파일이 존재하지 않는 경우
```

## 8. `get_required_tag()`

```python
def get_required_tag(parent: Tag, selector: str, field_name: str) -> Tag
```

### 역할

부모 태그에서 필수 하위 태그를 찾아 반환한다.

### 매개변수

- `parent` : `Tag`
- `selector` : `str`
- `field_name` : `str`

### 반환 자료형

```text
Tag
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
부모 태그에서 필수 하위 태그를 찾아 반환한다.

Args:
    parent:
        검색 기준이 되는 부모 HTML 태그

    selector:
        찾을 CSS 선택자

    field_name:
        오류 메시지에 표시할 필드명

Returns:
    선택자와 일치하는 첫 번째 HTML 태그

Raises:
    ValueError:
        필수 태그를 찾지 못한 경우
```

## 9. `parse_rating()`

```python
def parse_rating(rating_tag: Tag) -> tuple[str, int]
```

### 역할

평점 태그의 클래스에서 평점 단어와 숫자 평점을 추출한다.

### 매개변수

- `rating_tag` : `Tag`

### 반환 자료형

```text
tuple[str, int]
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
평점 태그의 클래스에서 평점 단어와 숫자 평점을 추출한다.

Args:
    rating_tag:
        star-rating 클래스가 있는 HTML 태그

Returns:
    평점 단어와 숫자 평점의 튜플

Raises:
    ValueError:
        One부터 Five까지의 평점 클래스를 찾지 못한 경우
```

## 10. `parse_book_item()`

```python
def parse_book_item(product: Tag, base_url: str) -> dict[str, str | int]
```

### 역할

도서 상품 HTML 요소 한 건에서 도서 정보를 추출한다.

### 매개변수

- `product` : `Tag`
- `base_url` : `str`

### 반환 자료형

```text
dict[str, str | int]
```

### 내부에서 호출하는 현재 단계 함수

- `get_required_tag()`
- `parse_rating()`

### 코드 Docstring

```text
도서 상품 HTML 요소 한 건에서 도서 정보를 추출한다.

Args:
    product:
        article.product_pod 도서 상품 요소

    base_url:
        상대 URL을 절대 URL로 변환할 기준 URL

Returns:
    도서 한 건의 파싱 결과 딕셔너리

Raises:
    ValueError:
        필수 태그나 필수 속성을 찾지 못한 경우
```

## 11. `parse_books()`

```python
def parse_books(html_content: bytes, source_page: int, source_url: str, source_file: str) -> pd.DataFrame
```

### 역할

한 페이지의 원본 HTML에서 모든 도서 정보를 파싱한다.

### 매개변수

- `html_content` : `bytes`
- `source_page` : `int`
- `source_url` : `str`
- `source_file` : `str`

### 반환 자료형

```text
pd.DataFrame
```

### 내부에서 호출하는 현재 단계 함수

- `parse_book_item()`

### 코드 Docstring

```text
한 페이지의 원본 HTML에서 모든 도서 정보를 파싱한다.

각 도서 정보에 원본 페이지 번호, URL, 파일명을 함께 저장한다.

Args:
    html_content:
        파싱할 원본 HTML 바이트 데이터

    source_page:
        원본 페이지 번호

    source_url:
        원본 페이지 URL

    source_file:
        원본 HTML 파일명

Returns:
    한 페이지의 도서 정보가 저장된 DataFrame

Raises:
    ValueError:
        도서 상품 요소를 찾지 못한 경우
```

## 12. `validate_books_dataframe()`

```python
def validate_books_dataframe(books_df: pd.DataFrame) -> int
```

### 역할

파싱 결과 DataFrame의 필수 데이터와 값의 범위를 검증한다.

### 매개변수

- `books_df` : `pd.DataFrame`

### 반환 자료형

```text
int
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
파싱 결과 DataFrame의 필수 데이터와 값의 범위를 검증한다.

Args:
    books_df:
        한 페이지의 도서 파싱 결과 DataFrame

Returns:
    중복된 상세 URL의 개수

Raises:
    ValueError:
        DataFrame이 비어 있거나 필수 컬럼, 결측값,
        평점, 페이지 번호에 문제가 있는 경우
```

## 13. `create_interim_batch_directory()`

```python
def create_interim_batch_directory(batch_name: str, directory: Path=INTERIM_DIR) -> Path
```

### 역할

원본 HTML 배치와 같은 이름의 interim 배치 폴더를 생성한다.

### 매개변수

- `batch_name` : `str`
- `directory` : `Path`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
원본 HTML 배치와 같은 이름의 interim 배치 폴더를 생성한다.

Args:
    batch_name:
        YYYYMMDD_HHMMSS 형식의 원본 HTML 배치 이름

    directory:
        interim 배치 폴더들이 저장되는 기본 경로

Returns:
    생성된 interim 배치 폴더 경로

Raises:
    ValueError:
        배치 이름이 지정한 형식과 일치하지 않는 경우
```

## 14. `save_parsed_csv()`

```python
def save_parsed_csv(books_df: pd.DataFrame, source_page: int, interim_batch_dir: Path) -> Path
```

### 역할

한 페이지의 파싱 결과를 interim 배치 폴더 안의 CSV 파일로 저장한다.

### 매개변수

- `books_df` : `pd.DataFrame`
- `source_page` : `int`
- `interim_batch_dir` : `Path`

### 반환 자료형

```text
Path
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
한 페이지의 파싱 결과를 interim 배치 폴더 안의 CSV 파일로 저장한다.

배치 폴더가 수집 시각을 관리하므로
CSV 파일명에는 페이지 번호와 처리 단계만 포함한다.

Args:
    books_df:
        한 페이지에서 파싱한 도서 DataFrame

    source_page:
        원본 페이지 번호

    interim_batch_dir:
        파싱 CSV 파일을 저장할 interim 배치 폴더

Returns:
    저장된 CSV 파일 경로
```

## 15. `verify_saved_csv()`

```python
def verify_saved_csv(parsed_file: Path, original_df: pd.DataFrame) -> pd.DataFrame
```

### 역할

저장한 CSV 파일을 다시 읽고 저장 전후의 행 수를 검증한다.

### 매개변수

- `parsed_file` : `Path`
- `original_df` : `pd.DataFrame`

### 반환 자료형

```text
pd.DataFrame
```

### 내부에서 호출하는 현재 단계 함수

- 없음

### 코드 Docstring

```text
저장한 CSV 파일을 다시 읽고 저장 전후의 행 수를 검증한다.

Args:
    parsed_file:
        저장한 CSV 파일 경로

    original_df:
        CSV 저장 전 원본 DataFrame

Returns:
    다시 읽은 CSV DataFrame

Raises:
    ValueError:
        CSV 저장 전후의 행 수가 다른 경우
```

## 16. `run_extract()`

```python
def run_extract(batch_dir: Path | None=None) -> list[Path]
```

### 역할

수집 배치의 원본 HTML을 파싱하고 같은 배치 이름의 interim 폴더에 저장한다.

### 매개변수

- `batch_dir` : `Path | None`

### 반환 자료형

```text
list[Path]
```

### 내부에서 호출하는 현재 단계 함수

- `parse_batch_directory_name()`
- `find_raw_html_files()`
- `create_interim_batch_directory()`
- `find_latest_batch_directory()`
- `parse_raw_file_name()`
- `load_raw_html()`
- `parse_books()`
- `validate_books_dataframe()`
- `save_parsed_csv()`
- `verify_saved_csv()`

### 코드 Docstring

```text
수집 배치의 원본 HTML을 파싱하고 같은 배치 이름의 interim 폴더에 저장한다.

Args:
    batch_dir:
        파싱할 원본 HTML 수집 배치 폴더

        값을 전달하지 않으면 data/raw/html 폴더에서
        가장 최근 수집 배치 폴더를 자동으로 찾는다.

Returns:
    페이지별로 저장된 CSV 파일 경로 목록

Raises:
    ValueError:
        페이지 내 중복 상세 URL이 있는 경우
```

## 처리 결과

```text
data/raw/html/20260809_233901/
            ↓
data/interim/20260809_233901/
├─ books_page_001_parsed.csv
├─ books_page_002_parsed.csv
└─ books_page_003_parsed.csv
```

## 핵심 실행 함수

```python
run_extract()
```

06_2 단계의 개별 탐색, 파싱, 검증, 저장 함수를 순서대로 연결합니다.
