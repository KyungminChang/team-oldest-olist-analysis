# Distribution Hub 통합 전처리 노트북 설명서

## 1. 문서 개요

이 문서는 [`integrated_preprocessing.ipynb`](./integrated_preprocessing.ipynb)의 목적, 설계 기준, 처리 단계, 생성 변수, 검증 로직, 저장 결과를 설명합니다.

통합 노트북은 다음 두 기존 노트북의 장점을 결합하여 만들었습니다.

| 기존 노트북 | 반영한 내용 |
| --- | --- |
| `../HY/전처리.ipynb` | 테이블별 점검, 주문 단위 집계, 날짜 이상치 점검, 금액 QA |
| `dataset_preprocessing.ipynb` | 구매 날짜 파생변수, 고객/판매자 지역 정보, 최종 병합 목적 |

목표는 Olist 이커머스 데이터를 Distribution Hub 및 배송 흐름 분석에 사용할 수 있는 형태로 안전하게 준비하는 것입니다.

## 2. 핵심 설계 원칙

### 2.1 최종 데이터셋의 기준 단위

최종 데이터셋 `olist_item_df`의 한 행은 **주문에 포함된 상품 1건**입니다.

```text
기준 키: order_id + order_item_id
```

예를 들어 주문 한 건에 상품이 세 개 포함되어 있다면 최종 데이터에도 세 행이 존재합니다. 물류 분석에서는 상품별 판매자, 가격, 배송비와 이동 경로를 확인해야 하므로 주문 단위보다 주문-상품 단위가 적절합니다.

### 2.2 단순 전체 조인을 피하는 이유

Olist 원본 테이블은 동일한 주문에 여러 행이 존재할 수 있습니다.

| 테이블 | 주문당 여러 행이 발생하는 이유 |
| --- | --- |
| `order_items` | 한 주문에 여러 상품이 포함될 수 있음 |
| `order_payments` | 카드와 바우처 등 결제 방식이 분할될 수 있음 |
| `order_reviews` | 일부 주문에 여러 리뷰 레코드가 존재할 수 있음 |
| `geolocation` | 동일 우편번호에 여러 좌표가 기록되어 있음 |

이 테이블들을 원본 그대로 모두 조인하면 한 주문의 행이 곱셈 형태로 늘어날 수 있습니다. 예를 들어 상품 2건과 결제 3건이 있는 주문을 직접 조인하면 상품 정보가 6행으로 늘어나 금액 합계와 건수 분석이 왜곡됩니다.

따라서 통합 노트북은 다음 방식으로 처리합니다.

1. `order_items`는 분석 기준 테이블로 유지합니다.
2. `order_payments`와 `order_reviews`는 `order_id`별로 먼저 집계합니다.
3. `geolocation`은 우편번호별로 먼저 집계합니다.
4. 집계된 테이블만 `order_items`에 결합합니다.
5. 조인에는 `validate="many_to_one"`을 적용하여 예상하지 못한 키 중복을 즉시 탐지합니다.

### 2.3 주문 상태의 보존

전처리 단계에서는 취소 주문 등 특정 상태를 바로 삭제하지 않습니다. 원본 상태를 보존해야 이후에 다음과 같은 분석 기준을 선택할 수 있기 때문입니다.

- 전체 주문 흐름 분석
- 배송 완료 주문만 사용하는 배송 성과 분석
- 취소 주문 원인 또는 지역 분포 분석

대신 배송 성과 분석용 데이터셋은 아래 조건으로 별도 생성합니다.

```python
is_delivered == True and anomaly_flag == 0
```

## 3. 입력 데이터

노트북은 `notebooks/Distribution_hub/SG/data` 아래에 있는 9개 CSV 파일을 읽습니다.

| 파일명 | 주요 내용 | 핵심 연결 키 |
| --- | --- | --- |
| `olist_customers_dataset.csv` | 고객 및 고객 주소 | `customer_id` |
| `olist_geolocation_dataset.csv` | 우편번호별 좌표 및 지역 | `geolocation_zip_code_prefix` |
| `olist_order_items_dataset.csv` | 주문 내 상품, 판매자, 가격, 배송비 | `order_id`, `product_id`, `seller_id` |
| `olist_order_payments_dataset.csv` | 주문 결제 내역 | `order_id` |
| `olist_order_reviews_dataset.csv` | 주문 리뷰 | `order_id` |
| `olist_orders_dataset.csv` | 주문 상태와 주문/배송 날짜 | `order_id`, `customer_id` |
| `olist_products_dataset.csv` | 상품 정보 및 카테고리 | `product_id` |
| `olist_sellers_dataset.csv` | 판매자 주소 | `seller_id` |
| `product_category_name_translation.csv` | 포르투갈어 카테고리 영문 번역 | `product_category_name` |

노트북은 프로젝트 루트에서 실행하는 경우와 `SG` 디렉터리 근처에서 실행하는 경우를 모두 지원하도록 입력 폴더 후보를 순서대로 확인합니다.

## 4. 처리 단계별 설명

### 4.1 환경 설정 및 출력 폴더 준비

첫 번째 코드 셀에서는 `pandas`와 `Path`를 불러오고 입력 데이터 폴더를 탐색합니다. 결과 파일은 입력 폴더 내부의 `processed` 폴더에 저장됩니다.

```text
입력: notebooks/Distribution_hub/SG/data/
출력: notebooks/Distribution_hub/SG/data/processed/
```

`processed` 폴더가 존재하지 않으면 실행 시 자동으로 생성됩니다.

### 4.2 원본 CSV 로드 및 기본 점검

전체 CSV를 DataFrame으로 읽고, 테이블별 행 수와 컬럼 수를 확인합니다. 이어서 주요 기준 키에 중복이 있는지와 전체 결측 셀 수를 확인합니다.

검사 대상 예시는 다음과 같습니다.

| 테이블 | 검사 키 | 기대 사항 |
| --- | --- | --- |
| `orders` | `order_id` | 주문당 한 행이어야 함 |
| `customers` | `customer_id` | 고객 주문 식별자당 한 행이어야 함 |
| `products` | `product_id` | 상품당 한 행이어야 함 |
| `sellers` | `seller_id` | 판매자당 한 행이어야 함 |

또한 `order_status` 분포를 표시하여 배송 완료, 취소, 배송 중 등 상태를 확인할 수 있도록 합니다.

### 4.3 주문 날짜 변환 및 배송 파생변수

`orders` 테이블을 `orders_clean`으로 복사한 뒤 주요 날짜 문자열을 `datetime` 타입으로 변환합니다.

| 원본 컬럼 | 의미 |
| --- | --- |
| `order_purchase_timestamp` | 고객 주문 시점 |
| `order_approved_at` | 결제 승인 시점 |
| `order_delivered_carrier_date` | 운송사 인계 시점 |
| `order_delivered_customer_date` | 고객 수령 시점 |
| `order_estimated_delivery_date` | 약속된 배송 예정일 |

구매 시점에서 생성하는 변수는 다음과 같습니다.

| 파생 컬럼 | 설명 | 분석 활용 예 |
| --- | --- | --- |
| `purchase_date` | 구매 날짜, 시간 제거 | 일별 주문 추세 |
| `purchase_year` | 구매 연도 | 연간 비교 |
| `purchase_month` | 구매 월 | 계절성 분석 |
| `purchase_weekday` | 구매 요일 | 요일별 수요 분석 |
| `purchase_hour` | 구매 시간대 | 주문 집중 시간대 분석 |

배송 품질 및 날짜 이상치 관련 변수는 다음과 같습니다.

| 파생 컬럼 | 설명 |
| --- | --- |
| `anomaly_carrier_before_approved` | 운송사 인계일이 결제 승인일보다 빠른지 여부 |
| `anomaly_customer_before_carrier` | 고객 수령일이 운송사 인계일보다 빠른지 여부 |
| `anomaly_flag` | 위 두 날짜 역전 조건 중 하나라도 만족하면 `1` |
| `delivery_days_all` | 주문부터 고객 수령까지 걸린 일수 |
| `delivery_days_clean` | 날짜 이상치가 없는 주문에 대해서만 유지한 배송 일수 |
| `is_late` | 실제 고객 수령일이 예정일보다 늦었는지 여부 |
| `is_delivered` | 주문 상태가 `delivered`인지 여부 |

`delivery_days_all`은 원본 사실을 유지하기 위한 변수이고, 배송 성과 집계에는 `delivery_days_clean` 사용을 권장합니다.

### 4.4 고객 및 판매자 지역 전처리

SG 기존 노트북에서 수행하던 브라질 주 정보 변환을 반복문 대신 매핑 딕셔너리로 처리합니다. 이 방식은 읽기 쉽고 실행 속도도 안정적입니다.

#### 주 이름 한글 변환

`STATE_KOR_MAP`은 주 코드에 대응하는 한글 명칭을 제공합니다.

| 원본 값 | 파생 값 예 |
| --- | --- |
| `SP` | `상파울루주` |
| `RJ` | `리우데자네이루주` |
| `MG` | `미나스제라이스주` |
| `PR` | `파라나주` |

생성 컬럼은 다음과 같습니다.

| 파생 컬럼 | 기준 |
| --- | --- |
| `customer_state_kor` | 고객 주 코드 |
| `seller_state_kor` | 판매자 주 코드 |

#### 권역 변환

`REGION_MAP`은 주 코드를 브라질의 5개 권역으로 변환합니다.

| 권역 | 포함 주 예 |
| --- | --- |
| 북부 | `AC`, `AM`, `PA`, `TO` |
| 북동부 | `BA`, `CE`, `PE`, `RN` |
| 중서부 | `DF`, `GO`, `MT`, `MS` |
| 남동부 | `SP`, `RJ`, `MG`, `ES` |
| 남부 | `PR`, `RS`, `SC` |

생성 컬럼은 다음과 같습니다.

| 파생 컬럼 | 설명 |
| --- | --- |
| `customer_region` | 고객 거주 권역 |
| `seller_region` | 판매자 거주 권역 |

#### 우편번호별 위치 집계

`geolocation`에는 같은 우편번호에 좌표가 여러 개 있을 수 있습니다. 이를 직접 조인하지 않고 우편번호별로 아래와 같이 축약합니다.

| 집계 컬럼 | 처리 방식 |
| --- | --- |
| 위도 (`geo_lat`) | 평균 |
| 경도 (`geo_lng`) | 평균 |
| 도시 (`geo_city`) | 최빈값 |
| 주 (`geo_state`) | 최빈값 |

이 집계 결과는 고객과 판매자에 각각 붙으며 접두사로 구분됩니다.

| 고객 위치 컬럼 | 판매자 위치 컬럼 |
| --- | --- |
| `customer_geo_lat` | `seller_geo_lat` |
| `customer_geo_lng` | `seller_geo_lng` |
| `customer_geo_city` | `seller_geo_city` |
| `customer_geo_state` | `seller_geo_state` |

판매자의 원본 `seller_state`와 우편번호 기반 `seller_geo_state`가 다를 경우, 원본 데이터를 수정하지 않고 `seller_state_geo_mismatch`라는 Boolean 검증 컬럼으로 남깁니다.

### 4.5 상품 카테고리 번역

`products`와 `product_category_name_translation`을 결합하여 상품 카테고리의 영문명을 추가합니다.

| 컬럼 | 의미 |
| --- | --- |
| `product_category_name` | 원본 포르투갈어 카테고리 |
| `product_category_name_english` | 분석 및 시각화를 위한 영문 카테고리 |

번역이 누락된 상품 수를 출력하여 카테고리 기반 분석 시 유의할 범위를 확인합니다.

### 4.6 결제 및 리뷰의 주문 단위 집계

#### 결제 집계: `pay_agg`

한 주문에 여러 결제 수단이나 순차 결제가 존재할 수 있으므로 `order_id` 기준으로 집계합니다.

| 생성 컬럼 | 집계 방식 | 의미 |
| --- | --- | --- |
| `payment_value_total` | 합계 | 주문별 총 결제 금액 |
| `payment_installments_max` | 최댓값 | 최대 할부 개월 수 |
| `payment_type_nunique` | 고유값 개수 | 사용 결제 수단 종류 수 |
| `payment_types` | 고유 결제 수단 문자열 결합 | 사용된 결제 방식 목록 |

#### 리뷰 집계: `rev_agg`

리뷰 날짜 컬럼을 `datetime`으로 변환한 뒤 주문 단위로 집계합니다.

| 생성 컬럼 | 집계 방식 | 의미 |
| --- | --- | --- |
| `review_score_mean` | 평균 | 주문별 평균 리뷰 점수 |
| `review_count` | 건수 | 주문에 연결된 리뷰 수 |
| `review_creation_min` | 최솟값 | 최초 리뷰 작성일 |
| `review_answer_max` | 최댓값 | 마지막 리뷰 응답일 |

두 집계 테이블 모두 집계 후 `order_id` 중복이 없는지 `assert`로 확인합니다.

### 4.7 최종 데이터셋 결합

최종 데이터셋은 아래 순서로 결합합니다.

```text
order_items
  + orders_clean
  + customers_clean
  + sellers_clean
  + products_clean
  + pay_agg
  + rev_agg
= olist_item_df
```

모든 조인은 `how="left"`를 사용합니다. 즉, 기준 데이터인 주문 상품 행을 유지하면서 연결 가능한 속성을 추가합니다.

또한 각 조인에 `validate="many_to_one"`을 지정합니다.

```python
.merge(orders_clean, on="order_id", how="left", validate="many_to_one")
```

이는 주문 상품 여러 행이 하나의 주문, 고객, 판매자, 상품 또는 주문별 집계 행에 연결되는 형태만 허용한다는 의미입니다. 대상 테이블의 키가 예기치 않게 중복되면 코드가 오류를 발생시켜 데이터 중복 확대를 방지합니다.

추가로 생성하는 물류 분석용 변수는 다음과 같습니다.

| 파생 컬럼 | 설명 | 활용 예 |
| --- | --- | --- |
| `item_total` | 상품 가격 + 상품 배송비 | 주문별 상품 총액 검증 |
| `seller_customer_same_state` | 판매자와 고객의 주가 동일한지 여부 | 주내/주간 배송 비교 |
| `delivery_route` | `판매자 주 -> 고객 주` 문자열 | 주요 배송 경로 분석 |

### 4.8 배송 분석용 데이터셋 생성

`delivered_item_df`는 배송 성능 분석에 바로 사용할 수 있도록 아래 조건으로 생성됩니다.

```python
olist_item_df["is_delivered"] & olist_item_df["anomaly_flag"].eq(0)
```

따라서 다음 데이터는 제외됩니다.

- 배송 완료 상태가 아닌 주문 상품
- 배송 날짜 순서가 비정상적인 주문 상품

이 데이터셋은 배송 소요일, 배송 지연률, 판매자-고객 권역별 배송 성능, 물류 허브 후보 지역 분석에 적합합니다.

## 5. 최종 QA 검증

노트북은 데이터를 저장하기 전에 다음 검증을 수행합니다.

### 5.1 조인 후 행 수 보존

```python
assert len(olist_item_df) == len(order_items)
```

최종 데이터셋의 행 수가 원본 `order_items`와 같아야 합니다. 다르면 조인 과정에서 행이 늘어나거나 사라진 것이므로 분석에 사용하기 전에 원인을 수정해야 합니다.

### 5.2 주문-상품 키 중복 확인

```python
assert not olist_item_df.duplicated(["order_id", "order_item_id"]).any()
```

`order_id`와 `order_item_id` 조합이 최종 데이터에서 중복되지 않아야 합니다.

### 5.3 주요 연결 컬럼 결측 확인

아래 컬럼의 결측 건수를 출력합니다.

| 컬럼 | 점검 목적 |
| --- | --- |
| `customer_id` | 주문과 고객의 연결 상태 |
| `seller_id` | 상품과 판매자의 연결 상태 |
| `product_id` | 상품 정보 연결 상태 |
| `payment_value_total` | 결제 집계 연결 상태 |
| `product_category_name_english` | 카테고리 번역 누락 상태 |

### 5.4 상품 금액과 결제 금액 비교

상품별 금액은 아래와 같이 계산됩니다.

```python
item_total = price + freight_value
```

이후 주문별 `item_total` 합계와 결제 총액 `payment_value_total`의 차이를 계산합니다.

```python
payment_minus_item = payment_value_total - item_total
```

절대 차이가 `1`을 초과하는 주문 수를 출력하여 결제와 상품 합계가 불일치하는 사례를 점검합니다. 이 차이는 데이터 오류뿐 아니라 할인, 정산 또는 데이터 정의 차이에서 발생할 수도 있으므로 무조건 삭제하기보다는 별도 조사 대상으로 다루는 것이 좋습니다.

## 6. 출력 파일

노트북의 마지막 셀을 실행하면 아래 두 파일이 생성됩니다.

| 출력 파일 | 포함 범위 | 권장 용도 |
| --- | --- | --- |
| `data/processed/olist_order_item_level.csv` | 모든 주문 상태의 주문-상품 데이터 | 주문 흐름, 취소 포함 전체 현황 분석 |
| `data/processed/olist_delivered_item_level.csv` | 완료 배송이며 날짜 이상치가 없는 주문-상품 데이터 | 배송 시간, 지연, 지역/허브 분석 |

저장 인코딩은 `utf-8-sig`입니다. 한글 파생 컬럼을 Excel 등에서 열 때 문자 깨짐을 줄이기 위한 설정입니다.

## 7. 실행 방법

프로젝트 루트에서 Jupyter 환경을 열고 아래 노트북을 순서대로 전체 실행합니다.

```text
notebooks/Distribution_hub/SG/integrated_preprocessing.ipynb
```

실행 전에 다음 입력 파일들이 `notebooks/Distribution_hub/SG/data` 폴더에 존재하는지 확인합니다.

```text
olist_customers_dataset.csv
olist_geolocation_dataset.csv
olist_order_items_dataset.csv
olist_order_payments_dataset.csv
olist_order_reviews_dataset.csv
olist_orders_dataset.csv
olist_products_dataset.csv
olist_sellers_dataset.csv
product_category_name_translation.csv
```

정상 실행 시 마지막 셀에서 출력 CSV의 저장 경로와 데이터프레임 크기를 확인할 수 있습니다.

## 8. 분석 활용 예시

### 8.1 배송 지연률 분석

배송 완료 데이터인 `olist_delivered_item_level.csv`를 사용하여 `is_late` 평균을 권역별 또는 배송 경로별로 계산할 수 있습니다.

추천 기준 컬럼:

- `seller_region`
- `customer_region`
- `delivery_route`
- `is_late`

### 8.2 배송 소요 시간 분석

`delivery_days_clean`을 사용하면 날짜 순서가 비정상인 레코드를 제외한 배송 기간을 분석할 수 있습니다.

추천 기준 컬럼:

- `delivery_days_clean`
- `seller_state`
- `customer_state`
- `product_category_name_english`

### 8.3 물류 허브 후보 탐색

판매자 출발 지역과 고객 도착 지역 사이의 물량 및 배송 시간을 비교하여, 물량이 많고 배송 시간이 긴 경로를 파악할 수 있습니다.

추천 기준 컬럼:

- `delivery_route`
- `seller_geo_lat`, `seller_geo_lng`
- `customer_geo_lat`, `customer_geo_lng`
- `delivery_days_clean`
- `freight_value`

### 8.4 시간대별 수요 변화 분석

구매시점 파생변수를 활용하여 물류 처리 수요가 집중되는 기간을 살펴볼 수 있습니다.

추천 기준 컬럼:

- `purchase_year`
- `purchase_month`
- `purchase_weekday`
- `purchase_hour`

## 9. 해석 시 주의사항

1. 최종 데이터는 주문이 아니라 주문-상품 단위입니다. 주문 수를 계산할 때는 `order_id.nunique()`를 사용해야 합니다.
2. `payment_value_total`은 주문 단위 값이므로, 주문에 상품이 여러 개 있는 최종 데이터에서 단순 합계하면 결제액이 중복 합산됩니다. 결제액 분석 시 주문별 한 행으로 축약한 뒤 합산해야 합니다.
3. `review_score_mean`도 주문 단위 집계 값으로 각 상품 행에 반복됩니다. 상품 수 기준 평균과 주문 수 기준 평균은 다를 수 있습니다.
4. `seller_state_geo_mismatch`는 데이터 오류가 확정되었다는 뜻이 아니라, 원본 판매자 주와 우편번호 기반 대표 주가 다름을 표시한 검토 대상 플래그입니다.
5. `olist_delivered_item_level.csv`는 배송 성과 분석을 위한 정제 데이터로, 취소 및 미완료 주문 분석에는 전체 데이터 파일을 사용해야 합니다.

## 10. 향후 확장 아이디어

- 판매자와 고객 좌표를 이용한 직선거리 계산 및 거리 대비 배송시간 비교
- 주문별 데이터셋을 별도로 생성하여 매출 및 결제 분석에 사용
- 상품 부피 및 무게를 활용한 배송비 효율 분석
- 경로별 물량, 지연률, 배송비를 결합한 물류 허브 후보 점수화
- 전처리 결과를 자동 확인하는 별도 테스트 스크립트 또는 데이터 품질 리포트 추가
