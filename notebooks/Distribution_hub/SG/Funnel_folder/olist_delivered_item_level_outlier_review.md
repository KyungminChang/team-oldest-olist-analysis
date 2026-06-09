# olist_delivered_item_level.csv 이상치 검토

대상 파일: `data/processed/olist_delivered_item_level.csv`  
행/열: 108,581행 x 67열

## 결론

명백히 제거해야 할 행은 거의 없습니다. 다만 아래 값들은 분석 목적에 따라 대체, 플래그 처리, 또는 상위 꼬리값 보정이 필요합니다.

가장 우선적으로 처리할 후보는 `product_weight_g = 0`, `payment_installments_max = 0`, `seller_state_geo_mismatch = True`입니다. 가격, 배송비, 배송일수, 중량/부피의 상위 극단값은 실제 거래일 가능성이 높으므로 원본 행 삭제보다는 로그 변환, winsorizing, 별도 플래그를 권장합니다.

## 우선 처리 권장 항목

| 컬럼/조건 | 건수 | 비율 | 판단 | 권장 처리 |
|---|---:|---:|---|---|
| `product_weight_g = 0` | 8 | 0.007% | 물리적으로 불가능한 상품 중량 | 같은 `product_id` 또는 카테고리 중앙값으로 대체. 원본 추적용 플래그 추가 |
| `payment_installments_max = 0` | 2 | 0.002% | `credit_card`인데 할부 수 0은 비정상 | 1로 대체하거나 원천 payment 데이터에서 재집계 |
| `seller_state_geo_mismatch = True` | 619 | 0.570% | 판매자 주와 지오코딩 주가 불일치 | 행 삭제보다는 지오 컬럼을 결측 처리하거나 `seller_state` 기준으로 재지오코딩. 거리/경로 분석에서는 주의 |
| `product_width_cm > 105` 또는 `dim_sum > 200` | 8 | 0.007% | 택배 규격 기준으로 의심 가능 | 배송비/부피 모델에서는 플래그 처리 후 winsorizing 또는 원천 확인 |

## 삭제보다는 보정이 나은 항목

| 컬럼/조건 | 건수 | 비율 | 판단 | 권장 처리 |
|---|---:|---:|---|---|
| `delivery_days_clean > 81` | 107 | 0.099% | 배송 지연의 극단값. 최대 209일 | 지연 분석에서는 유지. 평균 배송일/모델링에서는 p99.5 또는 p99.9 winsorizing |
| `delivery_days_clean >= 180` | 16 | 0.015% | 매우 긴 배송 지연 | 별도 `very_late` 플래그 권장. 삭제하면 장기 지연 리스크가 사라짐 |
| `price > 1200` | 534 | 0.492% | 고가 상품 꼬리값 | 매출/객단가 분석에서는 유지. 회귀/시각화에서는 로그 변환 또는 p99.5 winsorizing |
| `price > 2065.934` | 109 | 0.100% | 최상위 0.1% | 삭제보다 민감도 분석 권장 |
| `freight_value > 105.321` | 543 | 0.500% | 고배송비 꼬리값 | 중량/거리 영향일 수 있음. 로그 변환 또는 p99.5 winsorizing |
| `freight_value > 175.6` | 109 | 0.100% | 최상위 0.1% | 배송비 모델에서는 영향이 커서 별도 플래그 권장 |
| `payment_value_total > 1655.774` | 543 | 0.500% | 결제 총액 상위 꼬리값 | item-level 데이터에서는 주문 총액이 반복될 수 있으므로 제거 기준으로 쓰지 않는 편이 안전 |
| `product_weight_g > 30000` | 3 | 0.003% | 30kg 초과로 의심 | 원천 확인 전 삭제 비추천. 배송비 모델에서는 상한 처리 가능 |
| `product_weight_g > 22900` | 538 | 0.495% | 중량 상위 0.5% | 대형 상품일 가능성. 플래그 또는 winsorizing |
| `volume_cm3 > p99.5` | 536 | 0.494% | 부피 상위 꼬리값 | 배송비/리드타임 모델에서 플래그 또는 winsorizing |

## 유지 권장 항목

| 컬럼/조건 | 건수 | 비율 | 판단 |
|---|---:|---:|---|
| `freight_value = 0` | 370 | 0.341% | 무료배송/프로모션 가능성이 있어 결측이나 오류로 단정하기 어려움 |
| `delivery_days_clean = 0` | 16 | 0.015% | 당일 배송 또는 날짜 단위 반올림 결과일 수 있음 |
| 가격 최댓값 6,735 | 1 | 0.001% 미만 | `housewares`, `computers`, `art` 등 고가 카테고리에 분포. 즉시 삭제 근거 부족 |
| 배송비 최댓값 409.68 | 1 | 0.001% 미만 | 무거운 상품/장거리 배송과 함께 나타남 |

## 주요 분포 기준

| 컬럼 | 중앙값 | p95 | p99 | p99.5 | p99.9 | 최대 |
|---|---:|---:|---:|---:|---:|---:|
| `price` | 74.90 | 349.65 | 889.00 | 1200.00 | 2065.93 | 6735.00 |
| `freight_value` | 16.25 | 45.09 | 83.83 | 105.32 | 175.60 | 409.68 |
| `item_total` | 92.15 | 376.56 | 921.05 | 1290.37 | 2150.47 | 6929.31 |
| `payment_value_total` | 114.34 | 528.78 | 1230.00 | 1655.77 | 2754.90 | 13664.08 |
| `delivery_days_clean` | 10 | 29 | 45 | 53 | 81 | 209 |
| `product_weight_g` | 700 | 9750 | 18250 | 22900 | 30000 | 40425 |
| `volume_cm3` | - | 57600 | 111150 | 137677 | 246970 | 296208 |

## 예시 극단값

### 가격 상위 예시

| `order_id` | 카테고리 | `price` | `freight_value` | `delivery_days_clean` | `product_weight_g` |
|---|---|---:|---:|---:|---:|
| `0812eb902a67711a1cb742b3cdaa65ae` | housewares | 6735.00 | 194.31 | 18 | 30000 |
| `fefacc66af859508bf1a7934eab1e97f` | computers | 6729.00 | 193.21 | 20 | 5660 |
| `f5136e38d1a14a4dbd87dff67da82701` | art | 6499.00 | 227.66 | 11 | 7400 |

### 배송일수 상위 예시

| `order_id` | 카테고리 | `delivery_days_clean` | `is_late` | 출발/도착 주 |
|---|---|---:|---|---|
| `ca07593549f1816d26a572e06dc1eab6` | auto | 209 | True | MG -> ES |
| `1b3190b2dfa9d789e1f14c05b647a14a` | cool_stuff | 208 | True | SP -> RJ |
| `440d0d17af552815d15a9e41abe49359` | consoles_games | 195 | True | MG -> PA |

## 추천 처리안

1. 원본 CSV는 그대로 유지한다.
2. 분석용 파생 데이터에서 아래 플래그를 추가한다.
   - `is_zero_weight`
   - `is_zero_installments`
   - `is_seller_geo_mismatch`
   - `is_extreme_delivery_p999`
   - `is_extreme_price_p995`
   - `is_extreme_freight_p995`
   - `is_extreme_weight_or_volume_p995`
3. 모델링/시각화용 수치 컬럼은 목적별로 처리한다.
   - 매출/가격 분석: `price`, `item_total`은 유지하되 로그 변환 검토
   - 배송비 모델: `freight_value`, `product_weight_g`, `volume_cm3`는 p99.5 winsorizing 후보
   - 배송 리드타임 평균/회귀: `delivery_days_clean`은 p99.5 또는 p99.9 winsorizing 후보
   - 지연 리스크 분석: 긴 배송일수는 제거하지 말고 유지
4. 명백 오류만 대체한다.
   - `product_weight_g = 0`: 카테고리 또는 상품 단위 중앙값 대체
   - `payment_installments_max = 0`: 1 또는 원천 payment 재집계값으로 대체
   - 판매자 지오 불일치: 좌표/geo state를 재검증하거나 결측 처리

## 한 줄 판단

이 데이터에서 삭제 후보는 거의 없고, `중량 0`, `할부 수 0`, `판매자 지오 불일치`는 대체/정정 후보입니다. 가격, 배송비, 배송일수, 중량/부피 상위값은 실제 비즈니스 현상일 가능성이 높으므로 삭제보다 플래그와 winsorizing이 더 안전합니다.

## 생성한 정정 CSV

정정 파일: `data/processed/olist_delivered_item_level_cleaned_errors.csv`

원본 행은 삭제하지 않았고, 명백 오류만 최소 수정했습니다.

| 처리 항목 | 처리 방식 | 처리 건수 |
|---|---|---:|
| `product_weight_g = 0` | 같은 `product_id`의 정상 중량이 없어 `product_category_name_english = bed_bath_table` 중앙값 `1300g`로 대체 | 8 |
| `payment_installments_max = 0` | `credit_card` 결제의 최소 정상 할부 수로 보고 `1`로 대체 | 2 |
| `seller_state_geo_mismatch = True` | 좌표를 임의 대체하지 않고 `is_seller_geo_unreliable` 플래그만 추가 | 619 |

추가 컬럼은 `is_seller_geo_unreliable`만 남겼습니다.

- `is_seller_geo_unreliable`: 판매자 주(`seller_state`)와 지오코딩 주(`seller_geo_state`)가 불일치하는 행 표시

정정 후 `product_weight_g = 0`과 `payment_installments_max = 0`은 0건입니다.
