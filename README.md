# Smart Factory Power Forecasting

> 스마트 팩토리의 모듈별 전력 소비 패턴을 분석하고, 모듈 군집과 공정 전체 예측을 결합해 전력 수요 및 비효율 구간을 분석한 프로젝트.

## Overview

본 프로젝트는 모듈별 전력 소비 특성의 차이를 하나의 모델로 처리할 때 발생하는 한계를 줄이기 위해 다음 흐름으로 진행.

1. 모듈별 전력 소비 패턴 EDA
2. 전력·전압·전류·역률 기반 파생변수 생성
3. K-means 기반 모듈 군집화
4. 공정 전체 / 클러스터 단위 예측 모델 비교
5. 두 예측값을 결합한 계층적 앙상블
6. 예측 결과를 활용한 비효율 시간대 및 비용 절감 시나리오 분석

## Data & Preprocessing

원시 데이터는 고빈도 전력 센서 데이터로 구성.

- `operation`: 전 구간에서 1인 상수 변수로 확인되어 제거
- `localtime` 기반 datetime 처리
- lag / rolling / 시간대·요일 피처 생성
- 누적 유효전력 차분을 이용해 단위 시간 전력 사용량 파생

초기 EDA에서는 5분·15분·1시간 등 다양한 집계 단위를 검토.
5분은 노이즈가 과도하고 1시간은 피크를 지나치게 평활화하는 문제가 있어 15분 단위가 EDA 단계에서 유용한 후보로 검토되었고, **최종 팀 모델링 파이프라인에서는 10분 단위 재집계**를 사용.

## EDA Findings

분석 과정에서 반복적으로 관찰된 패턴은 다음과 같음.

- **2호기·4호기**: 평일 저녁과 휴일 새벽에 유사한 피크가 나타남
- **3호기**: 평일/휴일 차이는 작지만 월별 변동성이 큰 가변 공정 성격
- **6호기**: 평일 오전과 휴일 야간의 패턴 차이가 크고 일요일 피크가 자주 나타남
- 유효전력만 보지 않고 지연 무효전력, 역률, 에너지 사용량을 함께 비교

이러한 설비 해석은 실제 공정 메타데이터로 검증된 명칭이 아니라 **EDA에서 관찰한 패턴에 대한 분석 가설**.

자세한 해석은 [EDA findings](./docs/eda_findings.md)에 정리.

## Feature Engineering

### Power
- `10min_pow`
- `std_activePow`
- lagged active power
- change / rolling statistics

### Voltage & Current
- phase별 최소 전압
- phase별 최대·최소 전류
- `current_change_acc`
- `current_CV`
- `current_unbal_ratio`
- `current_Total`
- `voltage_CV`

### Power Quality
- time-weighted power factor
- time-weighted reactive power
- `pf_sign`
- `total_pf`

## Module Clustering

전력 품질 변수와 시간대별 패턴을 결합해 K-means clustering을 수행.

| Cluster | Modules | Interpretation |
|---|---|---|
| 0 | 2호기, 우측분전반2 | 기저 부하형 |
| 1 | 3호기, 예비건조기, 6호기, 분쇄기1, 좌측분전반 | 작업 동기·고변동형 |
| 2 | PM-3, 4호기, 호이스트, 분쇄기2 | 기동 부하형 |
| 3 | 우측분전반1, L-1 전등 | 환경·사용자 의존형 |

군집화는 단순 시각화가 아니라, 서로 다른 부하 특성을 가진 모듈에 동일한 모델을 적용하는 문제를 줄이기 위한 모델 설계 단계로 활용.

## Modeling Strategy

**bottom-up ensemble**이 핵심 전략으로 정리.

1. 모듈 또는 클러스터 단위 예측
2. 공장 전체 예측 모델을 별도로 구축
3. 두 예측 결과를 비교하고 가중 결합

### Validation
- 학습 구간: 2024-12 ~ 2025-04
- 검증 구간: 마지막 약 1주
- 주요 지표: MAE, RMSE, SMAPE

## Final Team Result

- Full-process model: SMAPE 약 **0.18%**, RMSE 약 **1.09**, MAE 약 **0.88**
- Cluster-level LightGBM: 종합 SMAPE 약 **0.18%**
- Final ensemble: **cluster : full-process = 2 : 8**

세부 모델링 결과는 [final modeling summary](./docs/final_modeling_summary.md)에 정리.

## Operational Analysis

최종 예측 결과를 이용해 2025년 5월 운영 시나리오를 분석.

- 예상 총 전력 사용량: 약 **2,607만 kWh**
- 예상 전기요금: 약 **46.9억 원**
- 예상 탄소배출량: 약 **11,038,955 kgCO₂**
- 비효율 의심 구간: **352시간**

심야 또는 휴일에도 높은 기저부하가 유지되는 구간의 전력을 5% 감축한다는 가정에서는:

- 월 예상 절감액: 약 **1억 2,291만 원**
- 연 환산 예상 절감액: 약 **14.7억 원**
- 월 CO₂ 절감 가능량: 약 **289톤**

> 위 수치는 실제 생산 스케줄·필수 기저부하·설비 제약을 검증하지 않은 시나리오 기반 추정치입니다. 실제 절감 가능액이 아니라 분석 결과를 경영 의사결정으로 연결한 예시로 해석.

## My Contribution

- MAE: **8.8554**
- RMSE: **13.4537**
- SMAPE: **1.8050%**

LSTM은 비교 실험으로 수행했으며 최종 팀 예측에는 트리 기반 모델이 사용.

## Notebooks

- [01_equipment_pattern_eda.ipynb](./notebooks/01_equipment_pattern_eda.ipynb) — 설비별 평일/휴일·요일·시간대 패턴 EDA
- [02_power_quality_eda.ipynb](./notebooks/02_power_quality_eda.ipynb) — 역률·무효전력·불평형·기저부하 EDA
- [03_module_clustering.ipynb](./notebooks/03_module_clustering.ipynb) — K-means 기반 설비 군집화
- [04_time_based_clustering_review.ipynb](./notebooks/04_time_based_clustering_review.ipynb) — 시간 패턴을 포함한 군집화 검토
- [05_modeling_experiments.ipynb](./notebooks/05_modeling_experiments.ipynb) — XGBoost / LightGBM / CatBoost / LSTM 비교 실험
- [06_cluster_modeling.ipynb](./notebooks/06_cluster_modeling.ipynb) — cluster별 CatBoost 모델링 실험

## Repository Structure

```text
.
├── README.md
├── notebooks/
│   ├── 01_equipment_pattern_eda.ipynb
│   ├── 02_power_quality_eda.ipynb
│   ├── 03_module_clustering.ipynb
│   ├── 04_time_based_clustering_review.ipynb
│   ├── 05_modeling_experiments.ipynb
│   └── 06_cluster_modeling.ipynb
├── docs/
│   ├── eda_findings.md
│   └── final_modeling_summary.md
└── data/
    └── README.md
```

## Tech Stack

`Python` `pandas` `NumPy` `matplotlib` `seaborn`  
`scikit-learn` `XGBoost` `LightGBM` `CatBoost` `Prophet` `PyTorch`
- 최종 팀 결과와 로컬 실험 결과를 구분해 기록했습니다.
- 비용·탄소 절감액은 가정 기반 시나리오입니다.
