# Smart Factory Power Forecasting

> 스마트 팩토리의 모듈별 전력 소비 패턴을 분석하고, 모듈 군집과 공정 전체 예측을 결합해 전력 수요 및 비효율 구간을 분석한 프로젝트입니다.

## Overview

본 프로젝트는 모듈별 전력 소비 특성의 차이를 하나의 모델로 처리할 때 발생하는 한계를 줄이기 위해 다음 흐름으로 진행했습니다.

1. 모듈별 전력 소비 패턴 EDA
2. 전력·전압·전류·역률 기반 파생변수 생성
3. K-means 기반 모듈 군집화
4. 공정 전체 / 클러스터 단위 예측 모델 비교
5. 두 예측값을 결합한 계층적 앙상블
6. 예측 결과를 활용한 비효율 시간대 및 비용 절감 시나리오 분석

## Data & Preprocessing

원시 데이터는 고빈도 전력 센서 데이터로 구성되어 있습니다.

- `operation`: 전 구간에서 1인 상수 변수로 확인되어 제거
- 시간 정보: `localtime` 기반 datetime 변환
- 고빈도 측정값을 더 안정적인 시간 단위로 재구성
- lag / rolling / 시간대·요일 피처 생성
- 누적 전력량 차분으로 에너지 사용량 변화량 계산

초기 EDA에서는 요일별·시간대별·평일/휴일 패턴을 비교했고, 설비별 소비 패턴 차이가 명확하게 존재함을 확인했습니다.

## EDA Findings

업로드된 EDA 자료에서는 다음 패턴이 반복적으로 확인됩니다.

- **2호기·4호기**: 평일 저녁과 휴일 새벽에 피크가 나타나는 유사한 패턴
- **3호기**: 평일/휴일 차이는 작지만 월별 변동성이 큰 가변 공정 성격
- **6호기**: 평일 오전과 휴일 야간의 패턴 차이가 크고, 일요일 피크가 자주 나타남
- 유효전력뿐 아니라 지연 무효전력과 에너지 사용량을 함께 보아 설비 상태를 해석

회의 자료에서는 모듈별로 동일한 시간 패턴을 가정하기보다 **모듈 특성에 맞춘 모델링 또는 군집 기반 접근**이 필요하다는 방향으로 발전했습니다.

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

K-means clustering을 통해 모듈을 다음 4개 유형으로 구분했습니다.

| Cluster | Modules | Interpretation |
|---|---|---|
| 0 | 2호기, 우측분전반2 | 기저 부하형 |
| 1 | 3호기, 예비건조기, 6호기, 분쇄기1, 좌측분전반 | 작업 동기·고변동형 |
| 2 | PM-3, 4호기, 호이스트, 분쇄기2 | 기동 부하형 |
| 3 | 우측분전반1, L-1 전등 | 환경·사용자 의존형 |

군집화는 단순 시각화가 아니라, 서로 다른 부하 특성을 가진 모듈에 동일한 모델을 적용하는 문제를 줄이기 위한 모델 설계 단계로 활용했습니다.

## Modeling Strategy

회의 기록에서는 **bottom-up ensemble**이 핵심 전략으로 정리되었습니다.

1. 모듈 또는 클러스터 단위 예측
2. 공장 전체 예측 모델을 별도로 구축
3. 두 예측 결과의 차이를 비교해 가중 결합

### Validation
- 학습 구간: 2024-12 ~ 2025-04
- 검증 구간: 마지막 약 1주
- 주요 지표: MAE, RMSE, SMAPE

## Local Experiment Results

업로드된 로컬 노트북에는 최종 제출 전 다양한 실험 결과가 남아 있습니다. 이 수치들은 **최종 제출 성능과 구분**해서 봐야 합니다.

| Experiment | Result |
|---|---:|
| XGBoost activePower baseline | SMAPE **0.7053%** |
| historical pattern 기반 XGBoost | SMAPE **0.9409%** |
| lagged sensor features XGBoost | SMAPE **0.7067%** |
| volatility-enhanced XGBoost | SMAPE **0.7070%** |
| hourly_pow XGBoost | SMAPE **1.7520%** |
| hourly_pow LightGBM | SMAPE **1.7556%** |
| hourly_pow CatBoost | SMAPE **1.7512%** |

LSTM도 비교 실험을 수행했지만 일부 로컬 실행에서는 NaN 문제가 발생해 최종 모델로 채택되지 않았습니다.

## Final Team Result

최종 팀 보고서에는 별도의 정리·재학습 결과가 기록되어 있습니다.

- Full-process model: SMAPE 약 **0.18%**, RMSE 약 **1.09**, MAE 약 **0.88**
- Cluster-level LightGBM: 종합 SMAPE 약 **0.18%**
- Final ensemble: **cluster : full-process = 2 : 8**

이 수치는 위의 로컬 실험 노트북 결과가 아니라 **최종 팀 제출 기준 결과**입니다.

## Operational Analysis

최종 예측 결과를 이용해 2025년 5월 운영 시나리오를 분석했습니다.

- 예상 총 전력 사용량: 약 **2,607만 kWh**
- 예상 전기요금: 약 **46.9억 원**
- 예상 탄소배출량: 약 **11,038,955 kgCO₂**
- 비효율 의심 구간: **352시간**

심야 또는 휴일에도 높은 기저부하가 유지되는 구간의 전력을 5% 감축한다는 가정에서는:

- 월 예상 절감액: 약 **1억 2,291만 원**
- 연 환산 예상 절감액: 약 **14.7억 원**
- 월 CO₂ 절감 가능량: 약 **289톤**

> 이 값은 운영 데이터와 MES/ERP 정보를 결합하지 않은 시나리오 기반 추정치입니다. 실제 절감 가능액으로 해석하기보다 분석 결과를 경영 의사결정으로 연결한 예시로 봐야 합니다.

## Code Materials

현재 보유한 로컬 분석 코드:

- `EDA호기~.ipynb` — 평일/휴일, 요일, 시간대별 모듈 패턴 EDA
- `EDA2.ipynb` — 전력 품질·기저부하·이상 패턴 EDA
- `클러스터링 분.ipynb` — 파생변수 기반 K-means 군집화 및 군집 해석
- `클러스터링 시간.ipynb` — 시간 특성을 포함한 군집화 실험
- `모델링.ipynb` — XGBoost / LightGBM / CatBoost / Prophet / LSTM 실험
- `클러스터 모델링.ipynb` — cluster별 CatBoost 모델링 실험

원본 노트북은 그래프 출력 때문에 파일 크기가 매우 커, GitHub에는 **코드와 마크다운은 유지하고 실행 output만 제거한 버전**으로 정리하는 것이 적절합니다.

## Repository Structure

```text
.
├── README.md
├── notebooks/
│   ├── 01_module_pattern_eda.ipynb
│   ├── 02_power_quality_eda.ipynb
│   ├── 03_module_clustering.ipynb
│   ├── 04_time_based_clustering_review.ipynb
│   ├── 05_modeling_experiments.ipynb
│   └── 06_cluster_modeling.ipynb
├── docs/
│   └── final_modeling_summary.md
└── data/
    └── README.md
```

## Tech Stack

`Python` `pandas` `NumPy` `matplotlib` `seaborn`  
`scikit-learn` `XGBoost` `LightGBM` `CatBoost` `Prophet` `PyTorch`

## Notes

- 원본 대회 데이터는 저장소에 포함하지 않습니다.
- 최종 팀 결과와 개인 로컬 실험 결과를 분리해 기록했습니다.
- 분석 과정에서 생성한 비용·탄소 절감액은 가정 기반 시나리오입니다.
