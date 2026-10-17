# Smart Factory Power Forecasting

> 스마트 팩토리의 모듈별 전력 소비 패턴을 분석하고, 계층적 예측 모델을 통해 공정 전체 전력 수요와 비효율 구간을 탐색한 프로젝트입니다.

## Overview

모듈별 전력 소비 특성의 차이를 하나의 모델로 처리할 때 발생하는 예측 오차를 줄이기 위해, **모듈 군집 단위 모델 + 공정 전체 모델**을 결합하는 계층적 접근을 설계했습니다.

프로젝트는 다음 세 단계로 진행했습니다.

1. 모듈별 전력 소비 패턴 및 전력 품질 특성 분석
2. K-means 기반 모듈 군집화와 전력 수요 예측
3. 예측 결과를 활용한 비효율 전력 구간 및 비용·탄소 절감 가능성 분석

## Data Processing

원시 데이터는 5초 단위 측정치로 구성되어 있어 단기 변동성이 매우 컸습니다. 구조적 패턴을 안정적으로 반영하기 위해 **10분 단위로 재집계**했습니다.

- `operation`: 모든 시점에서 1인 상수 변수로 확인되어 제거
- `timestamp`: `datetime` 기반 시계열 변수로 통합
- 주요 시계열 변수에 lagging 적용
- 누적 유효전력의 차분으로 `10min_pow` 생성

## Feature Engineering

### Power
- `10min_pow`
- `std_activePow`

### Voltage / Current
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
- `pf_sign`: 역률이 기준선 90% 아래로 내려간 횟수
- `total_pf`

## Module Clustering

모듈별 전력 소비 패턴을 기반으로 K-means clustering을 수행해 4개 유형으로 구분했습니다.

| Cluster Type | Characteristics |
|---|---|
| 기저 엔진형 | 조업 상태와 관계없이 기저부하가 지속되는 모듈 |
| 작업 동기형 | 유효·무효전력이 함께 움직이며 변동성이 큰 모듈 |
| 기동 부하형 | 초기 가동 시 전력이 급증한 뒤 안정되는 모듈 |
| 환경 의존형 | 사용자 개입 또는 외부 환경 영향으로 불규칙하게 작동하는 모듈 |

## Modeling

### Baseline
- Linear Regression
- SMAPE: **1.7912%**

### Full-Process Models
LightGBM, XGBoost, CatBoost를 비교했습니다.

최종 보고서 기준 세 모델 모두 약 다음 성능을 기록했습니다.

- SMAPE: **0.18%**
- RMSE: **1.09**
- MAE: **0.88**

피처 중요도에서 다양한 전력 변수를 상대적으로 고르게 반영한 **LightGBM**을 공정 전체 모델로 선정했습니다.

### Cluster-Level Models

클러스터 단위에서도 XGBoost, LightGBM, CatBoost를 비교했고 최종적으로 LightGBM을 선택했습니다.

| Cluster | SMAPE |
|---|---:|
| 0 | 0.184% |
| 1 | 0.1775% |
| 2 | 0.1814% |
| 3 | 0.179% |

종합 성능:
- SMAPE: **0.18%**
- MAE: **0.8946**
- RMSE: **1.1209**

### Final Ensemble

최종 모델은 **클러스터 단위 예측 : 공정 전체 예측 = 2 : 8** 비율로 결합했습니다.

## Operational Analysis

최종 예측값을 이용해 2025년 5월의 전력 소비와 운영 비효율 가능성을 분석했습니다.

- 예상 총 전력 사용량: 약 **2,607만 kWh**
- 예상 전기요금: 약 **46.9억 원**
- 예상 탄소배출량: 약 **11,038,955 kgCO₂**
- 비효율 의심 구간: **352시간**

심야 또는 주말에도 높은 기저부하가 유지되는 시간을 비효율 의심 구간으로 정의하고, 해당 구간의 전력 사용을 **5% 감축한다는 가정**을 적용했습니다.

- 월 예상 절감액: 약 **1억 2,291만 원**
- 연 환산 예상 절감액: 약 **14.7억 원**
- 월 CO₂ 절감 가능량: 약 **289톤**

> 위 절감액은 실제 공정의 필수 기저부하와 설비 운영 제약을 확인하지 않은 시나리오 기반 추정치입니다. 따라서 실무 적용 전에는 MES/ERP 및 실제 가동 스케줄을 함께 검증해야 합니다.

## My Contribution

프로젝트 최종 정리 자료에서 확인되는 개인 담당 영역은 **LSTM 기반 시계열 모델링 실험**입니다.

LSTM은 테스트 기준 다음 성능을 기록했습니다.

- MAE: **8.8554**
- RMSE: **13.4537**
- SMAPE: **1.8050%**

최종 예측에는 트리 기반 모델이 사용되었으며, LSTM 실험은 복잡한 시계열 모델이 항상 더 나은 성능을 보장하지 않는다는 비교 기준으로 활용했습니다.

## Repository Structure

```text
.
├── README.md
├── docs/
│   └── final_modeling_summary.md
└── data/
    └── README.md
```

## Tech Stack

`Python` `pandas` `NumPy` `scikit-learn`  
`LightGBM` `XGBoost` `CatBoost` `PyTorch` `K-means`

## Notes

- 본 저장소는 제4회 BDA 채용 공모전 스마트 팩토리 전력 소비 예측 프로젝트를 포트폴리오용으로 정리한 것입니다.
- 원본 프로젝트 노트북은 최종 팀 문서에서 파일명이 확인되지만 현재 연결된 Drive 경로에서는 직접 파일을 불러오지 못해, 확인 가능한 최종 분석 결과와 방법론을 먼저 정리했습니다.
