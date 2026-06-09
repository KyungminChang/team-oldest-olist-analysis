# 퍼널/코호트 마스터 테이블 문서

> **파일**: `data/funnel_master_all.csv` / `data/funnel_master_delivered.csv`  
> **생성 노트북**: `notebooks/개인의것/funnel_cohort_mastertable.ipynb`  
> **Grain**: 주문 1건 = 1행 (order grain) / 총 99,441행 / 45개 컬럼

---

## 1. 테이블 구성 개요

### 출력 파일 2종

| 파일명 | 행 수 | 용도 |
|--------|-------|------|
| `funnel_master_all.csv` | 99,441 | 퍼널 분석 (전체 order_status 포함) |
| `funnel_master_delivered.csv` | 96,478 | 코호트/리텐션 분석 (배송 완료 주문만) |

### 원본 소스 테이블 (9개)

| 원본 테이블 | 행 수 | 마스터 조인 방식 |
|------------|-------|----------------|
| `olist_orders_dataset` | 99,441 | 기준 테이블 |
| `olist_customers_dataset` | 99,441 | `customer_id` left join |
| `olist_order_items_dataset` | 112,650 | `order_id` 기준 **집계 후** left join |
| `olist_order_payments_dataset` | 103,886 | `order_id` 기준 **집계 후** left join |
| `olist_order_reviews_dataset` | 99,224 | 최신 1건 선택 후 `order_id` left join |
| `olist_products_dataset` | 32,951 | `product_id` → items에 먼저 조인 |
| `product_category_name_translation` | 71 | `product_category_name` → products에 먼저 조인 |
| `olist_sellers_dataset` | 3,095 | `seller_id` left join |
| `olist_geolocation_dataset` | 1,000,163 | `zip_prefix` 기준 **집계 후** left join (×2) |

---

## 2. 전처리 과정

### 2-1. 타임스탬프 변환

원본 데이터의 날짜 컬럼은 전부 `object(문자열)` 타입으로 저장되어 있어 `pd.to_datetime()`으로 변환.

| 변환 대상 컬럼 | 원본 타입 | 변환 후 |
|--------------|----------|---------|
| `order_purchase_timestamp` | object | datetime64 |
| `order_approved_at` | object | datetime64 |
| `order_delivered_carrier_date` | object | datetime64 |
| `order_delivered_customer_date` | object | datetime64 |
| `order_estimated_delivery_date` | object | datetime64 |
| `shipping_limit_date` (order_items) | object | datetime64 |
| `review_creation_date` | object | datetime64 |
| `review_answer_timestamp` | object | datetime64 |

### 2-2. 1:N 테이블 집계 (Grain 통일)

마스터 테이블의 grain은 **주문 1건 = 1행**이다.  
1:N 관계인 테이블을 직접 조인하면 행이 뻥튀기되므로 반드시 집계 후 조인.

#### order_items → items_agg

| 컬럼 | 집계 방식 | 비고 |
|------|----------|------|
| `item_count` | `order_item_id`의 max | 동일 주문 내 최대 번호 = 아이템 수 |
| `total_price` | `price`의 sum | |
| `total_freight` | `freight_value`의 sum | |
| `product_category` | 최빈값(mode) | 다중 카테고리 주문 시 가장 많은 카테고리 |
| `seller_id` | first | 첫 번째 아이템의 판매자 기준 |
| `shipping_limit_date` | min | 여러 아이템 중 **가장 이른** 기한 |
| `product_weight_g` | max | 가장 무거운 아이템 기준 |
| `product_volume_cm3` | max | 가장 큰 아이템의 부피 (별도 집계) |

#### payments → payments_agg

| 컬럼 | 집계 방식 | 비고 |
|------|----------|------|
| `total_payment_value` | sum | |
| `payment_type` | 최빈값(mode) | 주 결제 수단 |
| `payment_installments` | max | 최대 할부 개월 수 |
| `payment_type_nunique` | nunique | 결제 수단 종류 수 (복수 결제 탐지) |

#### reviews → reviews_dedup

- `order_id`당 중복 리뷰 **543건** 존재 (전체의 0.5%)
- `review_answer_timestamp` 기준 **내림차순 정렬 후 첫 번째** (가장 최근 답변 기준) 선택
- 보존 컬럼: `review_score`, `review_creation_date`, `review_answer_timestamp`
- `review_comment_title` / `review_comment_message` 는 결측률이 각각 88%, 59%로 높아 **제외**

#### geolocation → geo_agg

- 원본 100만 행을 직접 조인하면 행 수 폭증 위험
- `geolocation_zip_code_prefix` 기준으로 `lat`, `lng`를 **평균값으로 집계** (19,015행으로 축약)
- 고객 zip, 판매자 zip 각각 별도 left join

### 2-3. 조인 순서

```
orders
  ← customers       (customer_id)
  ← items_agg       (order_id)    ← products + category_tr 전처리 포함
  ← sellers         (seller_id)   ← items_agg에서 seller_id 추출
  ← payments_agg    (order_id)
  ← reviews_dedup   (order_id)
  ← geo_agg         (customer_zip_code_prefix)
  ← geo_agg         (seller_zip_code_prefix)
```

모든 조인은 `how='left'`이므로 **orders의 99,441행이 변하지 않음**.

---

## 3. 컬럼 상세 설명

### 3-1. 식별자

| 컬럼 | 타입 | 설명 | 비고 |
|------|------|------|------|
| `order_id` | string | 주문 고유 ID | PK, 결측 없음 |
| `customer_unique_id` | string | 고객 고유 ID | 재구매 추적용. `customer_id`와 달리 주문마다 재발급되지 않음 |
| `customer_id` | string | 해당 주문의 고객 ID | 주문마다 새로 발급. 재구매 분석에 사용 불가 |

> **주의**: 재구매·리텐션 분석은 반드시 `customer_unique_id` 기준으로 해야 한다.  
> `customer_id`는 동일 고객이라도 주문마다 값이 다르다.

---

### 3-2. 퍼널 타임스탬프 (원본)

퍼널 5단계의 원본 시각 데이터.

| 컬럼 | 타입 | 설명 | 결측 여부 |
|------|------|------|----------|
| `order_purchase_timestamp` | datetime | 주문 발생 시각 | 결측 없음 |
| `order_approved_at` | datetime | 결제 승인 시각 | 160건 결측 |
| `order_delivered_carrier_date` | datetime | 판매자→물류사 인도 시각 | 1,783건 결측 |
| `order_delivered_customer_date` | datetime | 물류사→고객 배송 완료 시각 | 2,965건 결측 |
| `order_estimated_delivery_date` | datetime | 주문 시 안내된 예상 배송일 | 결측 없음 |
| `shipping_limit_date` | datetime | 판매자가 물류사에 넘겨야 하는 기한 | order_items 기준 가장 이른 기한 |
| `order_status` | string | 주문 상태 | 8가지 값 존재 |

**order_status 값 목록**

| 값 | 건수 | 의미 |
|----|------|------|
| `delivered` | 96,478 | 배송 완료 |
| `shipped` | 1,107 | 배송 중 |
| `canceled` | 625 | 취소 |
| `unavailable` | 609 | 재고 없음 |
| `invoiced` | 314 | 청구서 발행 |
| `processing` | 301 | 처리 중 |
| `created` | 5 | 생성됨 |
| `approved` | 2 | 승인됨 |

---

### 3-3. 파생 컬럼 — 퍼널 구간 소요 시간

퍼널 5단계 사이의 소요 시간을 수치화한 컬럼들.

```
[구매] ──①──▶ [결제승인] ──②──▶ [물류사인도] ──③──▶ [고객수령] ──④──▶ [리뷰답변]
```

| 컬럼 | 단위 | 계산식 | 결측 조건 |
|------|------|--------|----------|
| `time_purchase_to_approved_h` | 시간(float) | `order_approved_at − order_purchase_timestamp` | `order_approved_at` 결측 시 |
| `time_approved_to_carrier_d` | 일(int) | `order_delivered_carrier_date − order_approved_at` | 두 컬럼 중 하나라도 결측 시 |
| `time_carrier_to_customer_d` | 일(int) | `order_delivered_customer_date − order_delivered_carrier_date` | 두 컬럼 중 하나라도 결측 시 |
| `total_lead_time_days` | 일(int) | `order_delivered_customer_date − order_purchase_timestamp` | `order_delivered_customer_date` 결측 시 |
| `review_response_days` | 일(int) | `review_answer_timestamp − review_creation_date` | 리뷰 없는 주문 시 |

> **구간 ②** (`time_approved_to_carrier_d`)가 음수인 경우 = 판매자가 승인 전에 이미 인도한 이상 케이스 → `anomaly_carrier_before_approved == 1`

---

### 3-4. 파생 컬럼 — 배송 지연

#### 고객 기준 지연 (최종 결과)

| 컬럼 | 타입 | 계산식 | 설명 |
|------|------|--------|------|
| `is_delayed` | Int8 (0/1/NaN) | `order_delivered_customer_date > order_estimated_delivery_date` | 1 = 지연, 0 = 정시/조기, NaN = 미배송 |
| `delay_days` | int/NaN | `order_delivered_customer_date − order_estimated_delivery_date` (일 수) | 양수 = 지연일, 음수 = 조기배송일, NaN = 미배송 |

실측값: `is_delayed == 1` → **7,827건 (8.1%)** (delivered 기준)

#### 판매자 귀책 지연 (구간 ② 책임 분리)

| 컬럼 | 타입 | 계산식 | 설명 |
|------|------|--------|------|
| `is_seller_late` | Int8 (0/1/NaN) | `order_delivered_carrier_date > shipping_limit_date` | 1 = 판매자가 약속 기한 내 물류사에 미인도 |
| `seller_delay_days` | int/NaN | `order_delivered_carrier_date − shipping_limit_date` (일 수) | 판매자 지연 일수 (양수 = 지연) |

실측값: `is_seller_late == 1` → **10,423건 (9.3%)** (전체 order_items 기준)

> **분석 포인트**: 최종 배송이 지연된 주문 중 `is_seller_late == 1`인 비율을 보면  
> 지연의 원인이 **판매자(구간 ②)** 때문인지 **택배사(구간 ③)** 때문인지 분리할 수 있다.

---

### 3-5. 파생 컬럼 — 이상치 플래그

날짜 역전(시간 순서 논리 위반) 케이스를 제거하지 않고 플래그로 관리.  
분석 시 `anomaly_flag == 0`으로 필터링하거나 포함/제외 비교에 사용.

| 컬럼 | 타입 | 계산식 | 실측 건수 |
|------|------|--------|---------|
| `anomaly_carrier_before_approved` | Int8 | `order_delivered_carrier_date < order_approved_at` | 1,359건 |
| `anomaly_customer_before_carrier` | Int8 | `order_delivered_customer_date < order_delivered_carrier_date` | 23건 |
| `anomaly_flag` | Int8 | 위 두 조건 중 하나라도 해당 | **1,382건 (1.4%)** |

> **주의**: 이상치를 단순 제거하면 배송 통계가 왜곡될 수 있다.  
> 배송 리드타임 분석 시 `anomaly_flag == 0` 필터를 명시적으로 적용할 것.

---

### 3-6. 고객 / 판매자 위치 정보

| 컬럼 | 설명 | 결측 |
|------|------|------|
| `customer_zip_code_prefix` | 고객 우편번호 앞 5자리 | 없음 |
| `customer_city` | 고객 도시명 | 없음 |
| `customer_state` | 고객 주(州) 코드 (2자리) | 없음 |
| `customer_region` | 고객 거주 지역 (5개 분류) | 없음 |
| `customer_lat` | 고객 우편번호 기준 평균 위도 | zip이 geolocation에 없을 때 |
| `customer_lng` | 고객 우편번호 기준 평균 경도 | zip이 geolocation에 없을 때 |
| `seller_id` | 판매자 고유 ID | 없음 |
| `seller_zip_code_prefix` | 판매자 우편번호 앞 5자리 | 없음 |
| `seller_city` | 판매자 도시명 | 없음 |
| `seller_state` | 판매자 주(州) 코드 | 없음 |
| `seller_region` | 판매자 거주 지역 (5개 분류) | 없음 |
| `seller_lat` | 판매자 우편번호 기준 평균 위도 | zip이 geolocation에 없을 때 |
| `seller_lng` | 판매자 우편번호 기준 평균 경도 | zip이 geolocation에 없을 때 |

**지역 분류 기준 (브라질 5개 지역)**

| region 값 | 해당 state |
|-----------|-----------|
| 남동부 | SP, MG, ES, RJ |
| 남부 | PR, SC, RS |
| 북동부 | BA, PE, CE, RN, PI, MA, SE, AL, PB |
| 중서부 | GO, MT, MS, DF |
| 북부 | AM, PA, AC, RO, RR, AP, TO |

**파생 컬럼**

| 컬럼 | 타입 | 계산식 | 설명 |
|------|------|--------|------|
| `is_same_state` | Int8 | `customer_state == seller_state` | 1 = 동일 주 내 배송 (로컬 배송) |

> **주의 (`seller_state`)**: 원본 데이터에서 `seller_state`가 `geolocation_state`와 불일치하는 **43건** 존재.  
> 지역 분석 정밀도가 중요한 경우 geolocation 기준으로 보정 고려.

---

### 3-7. 상품 정보

| 컬럼 | 설명 | 집계 방식 |
|------|------|----------|
| `item_count` | 주문 내 상품 수 | `order_item_id`의 max |
| `product_category` | 주문의 대표 카테고리 (영문) | 최빈값. 결측 시 NaN |
| `product_weight_g` | 주문 내 가장 무거운 상품의 무게(g) | max |
| `product_volume_cm3` | 주문 내 가장 큰 상품의 부피(cm³) | max (= 길이×높이×폭) |

---

### 3-8. 금액 정보

| 컬럼 | 설명 | 비고 |
|------|------|------|
| `total_price` | 주문 내 전체 상품가 합계 | |
| `total_freight` | 주문 내 전체 배송비 합계 | |
| `total_payment_value` | 실제 결제 금액 합계 | `total_price + total_freight`와 약간 차이 있을 수 있음 (249건, 1원 이상 차이) |

**파생 컬럼**

| 컬럼 | 타입 | 계산식 | 설명 |
|------|------|--------|------|
| `freight_ratio` | float | `total_freight / (total_price + total_freight)` | 배송비 비율. 0이면 무료 배송 |
| `price_per_item` | float | `total_price / item_count` | 평균 단가 (구매력 대리 지표) |
| `is_free_shipping` | Int8 | `total_freight == 0` | 1 = 무료 배송 (383건) |

---

### 3-9. 결제 정보

| 컬럼 | 설명 | 비고 |
|------|------|------|
| `payment_type` | 주 결제 수단 | 다중 결제 시 최빈값 |
| `payment_installments` | 최대 할부 개월 수 | 0~24 |
| `payment_type_nunique` | 결제 수단 종류 수 | 1 = 단일, 2 이상 = 복수 결제 |

**payment_type 값 분포**

| 값 | 건수 | 비율 | 특이사항 |
|----|------|------|---------|
| `credit_card` | 76,795 | 73.9% | |
| `boleto` | 19,784 | 19.0% | 브라질 은행전표. 영업일 소요 → 결제 승인 지연 가능성 |
| `voucher` | 5,775 | 5.6% | 쿠폰/포인트 사용 → A/B 테스트 시 대조군 오염 주의 |
| `debit_card` | 1,529 | 1.5% | |
| `not_defined` | 3 | 0.003% | 무시 가능 |

**파생 컬럼**

| 컬럼 | 타입 | 계산식 | 설명 |
|------|------|--------|------|
| `installment_bucket` | string | `payment_installments` 구간화 | `일시불` / `2~6회` / `7회이상` |

---

### 3-10. 리뷰 정보

| 컬럼 | 설명 | 결측 |
|------|------|------|
| `review_score` | 고객 만족도 점수 (1~5) | 리뷰 미작성 주문은 NaN |
| `review_creation_date` | 리뷰 작성 날짜 | 리뷰 없으면 NaN |
| `review_answer_timestamp` | 리뷰 답변 시각 | 리뷰 없으면 NaN |
| `review_response_days` | 리뷰 작성→답변 소요일 | 리뷰 없으면 NaN |

**review_score 분포**

| 점수 | 건수 | 비율 |
|------|------|------|
| 5 | 57,328 | 57.8% |
| 4 | 19,142 | 19.3% |
| 3 | 8,179 | 8.2% |
| 2 | 3,151 | 3.2% |
| 1 | 11,424 | 11.5% |

---

### 3-11. 주문 시간 특성

| 컬럼 | 타입 | 설명 | 값 예시 |
|------|------|------|---------|
| `purchase_year` | int | 주문 연도 | 2016, 2017, 2018 |
| `purchase_month` | int | 주문 월 | 1~12 |
| `purchase_yearmonth` | string | 주문 연월 (CSV 저장용 문자열) | `"2017-10"` |
| `purchase_dayofweek` | int | 주문 요일 숫자 | 0=월 ~ 6=일 |
| `purchase_dayofweek_name` | string | 주문 요일 이름 | `"Monday"` |
| `purchase_hour` | int | 주문 시간 | 0~23 |

> **주의**: `purchase_yearmonth`는 원본 연산 시 `pd.Period` 타입이나 CSV 저장 시 문자열로 변환됨.  
> 코호트 계산 시 `pd.Period`로 복원 필요: `pd.to_datetime(df['purchase_yearmonth']).dt.to_period('M')`

---

### 3-12. 코호트 분석 컬럼

리텐션 매트릭스 계산을 위한 핵심 컬럼 그룹.

| 컬럼 | 타입 | 계산식 | 설명 |
|------|------|--------|------|
| `purchase_order_rank` | int | `customer_unique_id`별 `order_purchase_timestamp` 오름차순 순위 | 1 = 첫 구매, 2 = 두 번째 구매 |
| `is_first_purchase` | int (0/1) | `purchase_order_rank == 1` | 1 = 해당 주문이 유저의 첫 구매 |
| `cohort_month` | string | 유저의 첫 구매 연월 | 리텐션 매트릭스의 행(row) 키 |
| `cohort_period_months` | int/NaN | `purchase_yearmonth - cohort_month` (개월 수) | 0 = 첫 구매월, 1 = 1개월 후, ... |

**활용 예시 — 리텐션 매트릭스**

```python
# delivered 주문만 사용
df_c = df[df['order_status'] == 'delivered'].copy()

cohort_matrix = (
    df_c
    .groupby(['cohort_month', 'cohort_period_months'])['customer_unique_id']
    .nunique()
    .unstack()
)

# 0개월 기준 비율로 변환
retention = cohort_matrix.divide(cohort_matrix[0], axis=0)
```

> **우측 절단(right-censoring) 주의**:  
> 데이터 기간이 2016.10~2018.10 (약 2년)으로 짧아, 최근 코호트일수록  
> 관측 기간이 부족해 리텐션이 실제보다 낮게 나온다.  
> 분석 시 최근 3~4개 코호트는 리텐션 해석에 주의.

---

## 4. 전체 결측치 현황 요약

| 결측률 수준 | 해당 컬럼 | 원인 |
|------------|----------|------|
| **~3%** | `order_delivered_customer_date`, `time_*` 관련 | 미배송 주문 (shipped, canceled 등) |
| **~3%** | `is_delayed`, `is_seller_late`, `delay_days` | 배송일 결측에 연쇄 |
| **~3%** | `customer_lat/lng`, `seller_lat/lng` | geolocation에 없는 우편번호 |
| **~2%** | `anomaly_flag`, `anomaly_*` | 배송일 결측에 연쇄 |
| **~2%** | `product_weight_g`, `product_volume_cm3` | 상품 정보 미입력 |
| **3~5%** | `review_score`, `review_creation_date` 등 | 리뷰 미작성 주문 |
| **1%** | `seller_lat/lng` | geolocation 불일치 |

---

## 5. 분석별 권장 필터

| 분석 목적 | 권장 필터 | 이유 |
|----------|----------|------|
| 퍼널 단계별 소요 시간 | `anomaly_flag == 0` 추가 권장 | 날짜 역전 1,382건 제외 |
| 배송 지연 분석 | `order_status == 'delivered'` | 미배송 주문은 지연 판단 불가 |
| 코호트/리텐션 | `order_status == 'delivered'` | `funnel_master_delivered.csv` 사용 권장 |
| 판매자 귀책 지연 | `is_seller_late` 컬럼 활용 | `shipping_limit_date` 기준 |
| 지역별 배송 분석 | `seller_lat.notna()` | geolocation 불일치 제거 |
| A/B 테스트 시뮬레이션 | `payment_type != 'voucher'` 고려 | 쿠폰 사용자 대조군 오염 방지 |

---

## 6. 주요 집계 수치 (참고)

| 지표 | 값 |
|------|-----|
| 전체 주문 수 | 99,441 |
| 배송 완료 주문 | 96,478 (97.0%) |
| 배송 지연 (고객 기준) | 7,827건 / 8.1% |
| 판매자 귀책 지연 | 10,423건 / 9.3% |
| 이상치(anomaly_flag) | 1,382건 / 1.4% |
| 고유 고객 수 | 96,096명 |
| 재구매 고객 수 | 2,997명 (3.1%) |
| 관찰 기간 | 2016-09 ~ 2018-10 |
| 리뷰 작성 주문 | 96,461건 (97.0%) |
