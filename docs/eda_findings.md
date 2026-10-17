# EDA Findings & Modeling Decisions

This document summarizes the main exploratory findings preserved in the project meeting notes and local notebooks.

## 1. Base-load behavior

Early EDA showed that average active power remained close to a stable baseline across weekdays, while reactive power showed relatively larger weekday-specific changes. The analysis therefore treated the factory as a mostly continuous process rather than a system with frequent full shutdowns.

The notes also observed that energy consumption did not fall to zero, supporting the interpretation that some equipment remained active throughout the day.

## 2. Time-of-day and weekday patterns

The project did not assume that identical hourly behavior applied to every piece of equipment.

Representative observations included:

- Units 2 and 4 showed similar workday/holiday timing patterns, with stronger holiday-morning activity in several views.
- Unit 3 showed relatively high month-to-month pattern variability and was treated as a more flexible / operator-sensitive process.
- Unit 6 showed a distinct night and Sunday pattern and was considered a possible support or post-process facility.

These interpretations were used as hypotheses for feature design and clustering, not as confirmed factory process labels.

## 3. Active vs. reactive power

Reactive power was retained because it helped distinguish different operating states that could look similar when only active power was considered.

The analysis used relationships among:

- active power
- lagging reactive power
- total power factor
- energy-use changes

to characterize module behavior and possible inefficient states.

## 4. Power-quality features

Exploratory notes considered the following engineering indicators:

- voltage unbalance
- current unbalance
- power factor
- current coefficient of variation
- phase-current imbalance
- current-change acceleration
- voltage coefficient of variation

The purpose was not only to improve predictive accuracy, but also to connect the model to interpretable operating conditions.

## 5. Aggregation-window experiments

During EDA, multiple smoothing / aggregation windows were discussed.

- 5-minute aggregation retained too much short-term noise.
- 1-hour aggregation could hide meaningful peaks.
- 15-minute windows appeared useful during exploratory peak analysis.

The final team modeling summary later adopted a 10-minute aggregation scheme. The repository therefore distinguishes exploratory window testing from the final modeling pipeline.

## 6. Why clustering was introduced

The meeting notes explicitly moved away from treating all modules as one homogeneous process.

The adopted strategy was:

1. characterize each module,
2. build module-level or pattern-level features,
3. cluster similar modules,
4. train cluster-level forecasting models,
5. compare the bottom-up result against a factory-level model,
6. combine both forecasts.

This evolved into the final bottom-up / full-process ensemble design.

## 7. Final cluster interpretation

The final clustering analysis grouped the modules as follows:

| Cluster | Modules | Interpretation |
|---|---|---|
| 0 | 2호기, 우측분전반2 | Base-load / persistent reactive-power pattern |
| 1 | 3호기, 예비건조기, 6호기, 분쇄기1, 좌측분전반 | Work-synchronous and high-variance pattern |
| 2 | PM-3, 4호기, 호이스트, 분쇄기2 | Startup-load pattern |
| 3 | 우측분전반1, L-1 전등 | Low-load / environment-dependent pattern |

These labels are analytical interpretations from observed patterns, not verified equipment-process metadata.

## 8. Modeling implication

The key project decision was that prediction should reflect both:

- **micro behavior**: cluster/module-level load characteristics
- **macro behavior**: total factory demand pattern

This led to the final ensemble in which the cluster-level forecast and full-process forecast were combined rather than relying on only one modeling scale.
