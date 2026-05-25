# 🛒 E-Commerce Operations Analytics: Funnel, Cohort, and A/B Testing
> **브라질 마켓플레이스 Olist 데이터를 활용한 주문-배송 오퍼레이션 최적화 및 유저 리텐션 분석 프로젝트**

본 프로젝트는 브라질의 대표적인 마켓플레이스 데이터셋인 **Olist 데이터**를 활용하여, 주문 생성부터 최종 배송 완료 및 리뷰 작성에 이르는 전 과정의 **비즈니스 퍼널(Funnel) 병목을 진단**하고, 배송 경험이 **고객 리텐션(Cohort)**에 미치는 영향을 추적하며, 가상의 오퍼레이션 개선안을 검증하기 위한 **A/B 테스트 설계**를 포함합니다.

---

## 📅 프로젝트 기간 & 참여도
* **기간:** 2026.0X ~ 2026.0X (약 X주)
* **담당 역할:** 데이터 전처리, 데이터 모델링(상관분석), 지표 정의, 시각화 대시보드 구축 및 리드미 작성 (100% 개인 프로젝트)

---

## 🛠️ Tech Stacks
* **Language:** `Python 3.10+`
* **Libraries:** `Pandas`, `NumPy`, `Matplotlib`, `Seaborn`, `SciPy` (Stats)
* **Environment:** `Jupyter Notebook` / `VS Code`

---

## 🏗️ 데이터 전처리 및 핵심 전략 (Data Preprocessing)
Olist 데이터의 특성과 한계를 극복하기 위해 타 도메인과 차별화된 전처리 파이프라인을 구축했습니다.

1. **시계열 데이터 모델링:** * 주문, 승인, 물류지 인도, 고객 수령 등 텍스트(`Object`)로 적힌 4가지 이상의 타임스탬프 컬럼을 `pd.to_datetime`으로 변환 및 정렬.
2. **Customer ID 이중 구조 해결:**
   * 트랜잭션마다 새로 발급되는 `customer_id` 대신, 고객 고유 식별자인 `customer_unique_id`를 매핑하여 유저 기준의 정확한 재구매율(Retention) 추적 환경 마련.
3. **오퍼레이션 파생 변수(Feature Engineering) 생성:**
   * 배송 한계일(`order_estimated_delivery_date`) 대비 실제 배송 완료일의 차이를 계산하여 배송 지연 여부 변수(`is_delayed`) 정의.
   * 주문부터 물류 인도까지의 소요 시간(`lead_time_carrier`) 계산으로 퍼널 세부 구간 구체화.

---

## 📊 분석 프레임워크 (Analysis Framework)

### 1. 퍼널 분석 (Funnel Analysis)
* **목적:** 주문 생성부터 최종 리뷰 작성까지의 유실률(Drop-off)과 소요 시간(Lead Time)을 측정하여 물류 병목 구간을 탐색합니다.
* **단계 설계:** `주문 완료 (Purchase)` ➡️ `결제 승인 (Approved)` ➡️ `물류 인도 (Carrier)` ➡️ `배송 완료 (Delivered)` ➡️ `리뷰 작성 (Review)`
* **핵심 질문:** "판매자가 택배사에 늦게 준 것인가, 택배사가 배송을 오래 한 것인가?"
### 2. 코호트 분석 (Cohort Analysis)
* **목적:** 유저의 첫 구매월(Cohort Month) 및 배송 만족도(Review Score)별 그룹화를 통해 시간 경과에 따른 재구매 패턴과 유저 유지율(Retention Rate)을 분석합니다.
* **가설 검증:** "첫 구매 시 배송 지연(`is_delayed == 1`)을 경험한 코호트 그룹은 정상 배송 그룹보다 리텐션이 유의미하게 낮을 것이다."
### 3. A/B 테스트 설계 (A/B Testing Simulation)
* **배경:** 배송 지연으로 인한 코호트 이탈을 막기 위해, **"배송 지연 유저 대상 선제적 알림 메커니즘 및 보상 쿠폰 발송 시스템"** 도입을 제안합니다.
* **실험 설계:**
  * **대조군 (Control):** 배송 지연 발생 시 별도 안내 없음 (기존 프로세스)
  * **실험군 (Variant):** 배송 지연 예측/확정 시 사과 알림 메시지 + 재구매 유도 쿠폰 즉시 발송
* **핵심 지표:** * **Primary Metric:** 실험 종료 후 30일 이내 재구매율 (Retention Rate)
  * **Secondary Metric:** 고객 만족도 점수 (Review Score)
* **통계적 검증:** `SciPy` 라이브러리를 활용한 두 그룹 간의 전환율 차이 검정 (귀무가설 기각 여부 확인)
---

## 📈 핵심 인사이트 및 비즈니스 제안
* **퍼널 병목:** 분석 결과, `[결제 승인 -> 물류 인도]` 구간보다 `[물류 인도 -> 고객 수령]` 구간에서 가장 큰 유실 및 지연이 발생함이 확인되었습니다. (특히 특정 주(State) 지역 집중 현상 발생)
* **코호트 결과:** 첫 구매 시 부정적 배송 경험(지연 및 낮은 리뷰)을 한 유저 코호트는 가입 2달 차 리텐션이 정상 그룹 대비 `X%` 감소하는 경향을 보였습니다.
* **선제적 조치 제안:** 데이터 기반의 선제적 알림 메커니즘을 도입하여 물류 페인포인트(Pain-point)를 서비스 관점에서 완화할 것을 제안합니다.

---
## 📂 프로젝트 폴더 구조
```text
├── data/                  # Olist 원본 데이터셋 (Git LFS 권장)
├── notebooks/
│   ├── 01_preprocessing.ipynb   # 데이터 병합 및 시계열 전처리
│   ├── 02_funnel_cohort.ipynb    # 퍼널/코호트 시각화 및 분석
│   └── 03_ab_testing.ipynb       # A/B 테스트 통계 검증 및 시뮬레이션
├── README.md              # 프로젝트 메인 설명서
└── requirements.txt       # 필요한 라이브러리 목록
