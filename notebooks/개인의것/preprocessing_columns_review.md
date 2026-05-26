# 전처리 컬럼 검토 — 퍼널/코호트 분석 보강 제안

> 검토 대상: `HY/전처리.ipynb`, `SG/dataset_preprocessing.ipynb`, `개인의것/sample.ipynb`, 원본 9개 CSV  
> 목적: 퍼널·코호트 분석에 유용하지만 현재 전처리에 빠진 컬럼 정리

---

## 1. 현재 내 전처리(`sample.ipynb`)에 있는 컬럼

| 컬럼 | 설명 |
|------|------|
| `is_delayed` | 배송 지연 여부 (0/1) |
| `delay_days` | 지연 일수 (음수 = 조기 배송) |
| `time_purchase_to_approved` | 주문→결제 승인 소요 시간(시간 단위) |
| `time_approved_to_carrier` | 결제 승인→물류사 인도 소요 일수 |
| `time_carrier_to_customer` | 물류사 인도→고객 수령 소요 일수 |
| `total_lead_time_days` | 주문→고객 수령 전체 리드타임(일) |
| `purchase_year/month/yearmonth` | 주문 연·월 |
| `purchase_dayofweek` | 주문 요일 (0=월요일) |
| `purchase_hour` | 주문 시간대 |
| `customer_state` | 고객 주(州) |
| `total_payment_value` | 주문 총 결제금액 |
| `payment_type` | 주 결제 수단 |
| `payment_installments` | 최대 할부 개월 수 |
| `item_count` | 주문 내 아이템 수 |
| `total_price` / `total_freight` | 상품가·운임 합계 |
| `product_category` | 상품 카테고리(영문) |
| `review_score` | 리뷰 점수 (1~5) |

---

## 2. HY 전처리에서 발견한 추가 아이디어

### 2-1. 날짜 역전 이상치 플래그 (`anomaly_flag`) ⭐ 권장

HY는 단순 음수 리드타임 제거 대신 **플래그로 관리**하는 방식 채택.  
실제로 날짜 역전이 발생하는 케이스가 **1,382건** 존재함.

```
carrier < approved  : 1,359건
customer < carrier  :    23건
```

| 컬럼 | 설명 | 분석 활용 |
|------|------|-----------|
| `anomaly_flag` | 날짜 역전 이상치 여부 (0/1) | 이상치 포함/제외 버전 비교 |
| `anomaly_carrier_before_approved` | 세부 플래그 | 판매자 처리 이상 구간 식별 |
| `anomaly_customer_before_carrier` | 세부 플래그 | 배송 구간 이상 식별 |
| `delivery_days_clean` | 이상치 제외 리드타임 | 왜곡 없는 배송 통계 |

> **적용 방법**: 현재 `negative_mask`로 제거하는 대신 플래그 컬럼을 만들어 분석 시 선택적으로 제외.

### 2-2. 리뷰 집계 방식 차이

HY는 `review_score_mean`(평균)과 `review_count`를 별도 집계.  
→ 현재 내 전처리는 최신 1건만 취하므로, 다중 리뷰 주문의 정보 손실 있음.

| 컬럼 | 설명 |
|------|------|
| `review_score_mean` | 주문당 리뷰 점수 평균 |
| `review_count` | 주문당 리뷰 수 |
| `review_creation_min` | 첫 리뷰 작성일 |

### 2-3. 결제 다양성 컬럼

| 컬럼 | 설명 | 분석 활용 |
|------|------|-----------|
| `payment_type_nunique` | 결제 수단 종류 수 | 복수 결제수단 사용 여부 |
| `payment_sequential_max` | 결제 순서 최대값 | 분할 결제 여부 확인 |

---

## 3. SG 전처리에서 발견한 추가 아이디어

### 3-1. 지역(Region) 컬럼 ⭐ 권장

SG는 주(State) → 브라질 5개 지역으로 매핑.  
퍼널 분석에서 **지역별 배송 병목** 파악에 직접 활용 가능.

| 컬럼 | 설명 | 분석 활용 |
|------|------|-----------|
| `customer_region` | 구매자 거주 지역 (남동부/남부/북동부/중서부/북부) | 지역별 배송 지연율 비교 |
| `seller_region` | 판매자 거주 지역 | 출발지별 리드타임 분석 |
| `seller_state` | 판매자 주(州) | 세부 물류 병목 지점 |

**지역 분류 기준:**

| 지역 | 해당 주(State) |
|------|----------------|
| 남동부 | SP, MG, ES, RJ |
| 남부 | PR, SC, RS |
| 북동부 | BA, PE, CE, RN, PI, MA, SE, AL, PB |
| 중서부 | GO, MT, MS, DF |
| 북부 | AM, PA, AC, RO, RR, AP, TO |

### 3-2. 시간 분해 방식

SG는 `purchase_date`와 `purchase_time`을 분리하고 `day_of_week`를 문자열로 저장.  
→ 내 전처리의 `purchase_dayofweek`(숫자)를 **문자열로도 추가**하면 시각화 시 편리.

| 컬럼 | 설명 |
|------|------|
| `purchase_date` | 날짜만 (date 타입) |
| `purchase_time_hour` | 시간대 (이미 있음: `purchase_hour`) |
| `day_of_week_name` | 요일 이름 (Monday~Sunday) |

---

## 4. 퍼널/코호트 분석을 위해 새로 추가할 컬럼 제안

현재 어떤 팀원도 만들지 않았지만 **분석 목적상 핵심**이 되는 파생 변수들.

### 4-1. 코호트 분석 핵심 ⭐⭐ 최우선

| 컬럼 | 계산 방식 | 분석 활용 |
|------|-----------|-----------|
| `purchase_order_rank` | `customer_unique_id`별 `order_purchase_timestamp` 순위 | 유저의 N번째 구매 식별 |
| `is_first_purchase` | `purchase_order_rank == 1` | 코호트 첫 구매월 정의 |
| `cohort_month` | 유저의 첫 구매 `purchase_yearmonth` | 코호트 그룹 키 |
| `cohort_period` | `purchase_yearmonth - cohort_month` (개월 수) | 리텐션 경과 개월 계산 |

```python
# 예시 코드
df['purchase_order_rank'] = (
    df.groupby('customer_unique_id')['order_purchase_timestamp']
    .rank(method='first')
    .astype(int)
)
df['is_first_purchase'] = (df['purchase_order_rank'] == 1).astype(int)

first_purchase = (
    df[df['is_first_purchase'] == 1]
    [['customer_unique_id', 'purchase_yearmonth']]
    .rename(columns={'purchase_yearmonth': 'cohort_month'})
)
df = df.merge(first_purchase, on='customer_unique_id', how='left')
df['cohort_period'] = (
    df['purchase_yearmonth'].dt.to_timestamp() -
    df['cohort_month'].dt.to_timestamp()
).dt.days // 30
```

### 4-2. 퍼널 분석 보강

| 컬럼 | 계산 방식 | 분석 활용 |
|------|-----------|-----------|
| `review_response_days` | `review_answer_timestamp - review_creation_date` | 퍼널 마지막 단계(리뷰→답변) 소요 |
| `seller_to_customer_distance` | 판매자-구매자 지역 동일 여부 | 주간 vs 도시간 배송 비교 |
| `is_same_state` | `customer_state == seller_state` | 로컬 배송 여부 |

### 4-3. 상품/금액 파생 변수

| 컬럼 | 계산 방식 | 분석 활용 |
|------|-----------|-----------|
| `freight_ratio` | `total_freight / (total_price + total_freight)` | 운임 부담 비율 (배송 지연 민감도) |
| `product_volume_cm3` | `length × height × width` | 상품 크기 → 배송 난이도 |
| `price_per_item` | `total_price / item_count` | 단가 (구매력 대리 지표) |

---

## 5. 원본 데이터 직접 분석 — 미활용 컬럼 발굴

### 5-1. `shipping_limit_date` (order_items) ⭐⭐⭐ 가장 중요한 발견

**판매자가 물류사에 넘겨야 하는 기한**으로, 실제 인도일(`order_delivered_carrier_date`)과 비교하면  
"배송 지연이 판매자 책임인지, 택배사 책임인지" 를 직접 분리할 수 있음.  
README의 핵심 질문과 정확히 일치하는 컬럼.

```
shipping_limit_date 위반 건수: 10,423건 / 전체 112,650건
위반율: 9.3%
```

| 컬럼 | 계산 방식 | 분석 활용 |
|------|-----------|-----------|
| `is_seller_late` | `order_delivered_carrier_date > shipping_limit_date` | 판매자 귀책 지연 여부 |
| `seller_delay_days` | `(carrier_date - shipping_limit_date).dt.days` | 판매자 지연 일수 |

→ **퍼널 `[승인→물류사 인도]` 구간 병목이 판매자/택배사 중 어디서 발생하는지 정량화 가능**

---

### 5-2. `payment_installments` (payments) — 구매력 대리 지표

```
1회(일시불): 52,546건 (50.6%)
2회:         12,413건
3회:         10,461건
10회:         5,328건
최대:        24회
```

| 컬럼 | 계산 방식 | 분석 활용 |
|------|-----------|-----------|
| `is_installment` | `payment_installments > 1` | 할부 사용 여부 (이미 포함) |
| `installment_bucket` | 1 / 2~6 / 7+ 구간 | 코호트 내 구매력 세그먼트 |

→ 할부 횟수가 많을수록 가격 민감 고객 → 배송 지연 시 이탈률 비교 가능

---

### 5-3. `payment_type` 분포 — 결제 수단별 행동 차이

```
credit_card:  76,795건 (73.9%)
boleto:       19,784건 (19.0%)  ← 브라질 특유의 은행전표 방식
voucher:       5,775건  (5.6%)
debit_card:    1,529건  (1.5%)
```

| 분석 포인트 | 내용 |
|------------|------|
| `boleto` 고객 | 결제 승인 지연 가능성 높음 (은행 영업일 의존) |
| `voucher` 고객 | 쿠폰/보상 사용자 → A/B 테스트 대조군 오염 주의 |

→ 코호트를 결제 수단별로 분리하면 `voucher` 사용자가 보상 쿠폰 실험과 겹치는지 확인 가능

---

### 5-4. `product_photos_qty` (products) — 상품 노출 품질

```
1장: 16,489개 (50%)
2장:  6,263개
평균:   2.2장 / 최대: 20장
```

| 컬럼 | 계산 방식 | 분석 활용 |
|------|-----------|-----------|
| `product_photos_qty` | 원본 그대로 | 사진 수 ↔ 리뷰 점수 상관 분석 |
| `has_single_photo` | `product_photos_qty == 1` | 낮은 노출 품질 플래그 |

---

### 5-5. `product_weight_g` + 부피 (products) — 물류 난이도

```
중앙값:   700g / 95th percentile: 10,850g
부피 중앙값: 6,840 cm³ / 95th: 63,369 cm³
```

| 컬럼 | 계산 방식 | 분석 활용 |
|------|-----------|-----------|
| `product_volume_cm3` | `length × height × width` | 부피 → 배송 난이도 |
| `is_heavy` | `product_weight_g > 10000` (95th) | 중량 화물 여부 |
| `is_bulky` | `volume_cm3 > 63369` (95th) | 대형 화물 여부 |

→ 무겁고 큰 상품일수록 `time_carrier_to_customer` 지연 여부 검증 가능

---

### 5-6. `review_response_days` — 퍼널 마지막 단계 소요

```
중앙값: 1일 / 75th: 3일 / 95th: 6일 / 최대: 518일
```

리뷰 작성 후 판매자/플랫폼 답변까지 소요 시간.  
퍼널의 `[리뷰 작성 → 답변]` 단계를 수치화하는 데 사용.

---

### 5-7. 재구매율 실측값 — 코호트 분석 기준선

```
1회 구매:  93,099명 (96.9%)
2회 구매:   2,745명  (2.9%)
3회 이상:    252명  (0.3%)
전체 재구매율: 3.1%
```

> **주의**: 데이터 기간(2016.10 ~ 2018.10, 약 2년)이 짧아 재구매율이 실제보다 낮게 관측됨.  
> 코호트 분석 시 데이터 수집 종료 시점에 가까운 코호트일수록 관측 기간이 짧아 리텐션이 낮게 나오는 **우측 절단(right-censoring)** 현상 고려 필요.

---

### 5-8. `freight_value == 0` — 무료 배송 케이스

```
freight_value == 0: 383건 (전체의 0.3%)
```

| 컬럼 | 계산 방식 | 분석 활용 |
|------|-----------|-----------|
| `is_free_shipping` | `total_freight == 0` | 무료 배송 ↔ 배송 지연율 비교 |

---

## 6. 우선순위 정리 (전체 통합)

| 우선순위 | 출처 | 컬럼 | 근거 |
|---------|------|------|------|
| ⭐⭐⭐ 필수 | 파생 | `purchase_order_rank`, `is_first_purchase`, `cohort_month`, `cohort_period` | 코호트 분석 자체가 불가 |
| ⭐⭐⭐ 필수 | 원본 `order_items` | `is_seller_late`, `seller_delay_days` | 퍼널 핵심 질문 "판매자 vs 택배사" 직접 분리 가능 |
| ⭐⭐ 권장 | HY 방식 | `anomaly_flag` | 이상치 1,382건 처리 방식 개선 |
| ⭐⭐ 권장 | SG 방식 | `customer_region`, `seller_region`, `seller_state` | 지역별 퍼널 병목 분석 핵심 |
| ⭐⭐ 권장 | 원본 `payments` | `installment_bucket` | 구매력 세그먼트 → 코호트 그룹화 |
| ⭐ 선택 | 파생 | `freight_ratio`, `is_same_state`, `review_response_days` | 추가 인사이트 |
| ⭐ 선택 | 원본 `products` | `product_volume_cm3`, `is_heavy`, `product_photos_qty` | 상품 특성 ↔ 배송 지연 상관분석 |
| ⭐ 선택 | 원본 `payments` | `is_free_shipping` | 무료 배송 효과 분석 |
| ⭐ 선택 | 시각화용 | `day_of_week_name`, `payment_type_nunique` | 탐색적 분석 |

---

## 6. 현재 전처리의 주의사항

1. **`carrier < approved` 1,359건**: 현재 `negative_mask`가 이를 잡지 못할 수 있음.  
   → `time_approved_to_carrier < 0`으로 필터링되어야 하지만, 두 컬럼 모두 결측이면 NaN이 되어 통과됨.  
   → HY 방식의 `anomaly_flag` 도입 권장.

2. **코호트 분석에 `purchase_yearmonth`(Period 타입)**: CSV 저장 시 문자열 변환 필요 (이미 처리).  
   → 코호트 계산 시 다시 `pd.Period`로 복원해야 함.

3. **SG의 `seller_state` 오류**: 원본 데이터의 `seller_state`가 geolocation과 불일치하는 **43건** 존재.  
   → 지역 분석 시 `seller_state`를 그대로 쓰면 오염될 수 있으므로 주의.
