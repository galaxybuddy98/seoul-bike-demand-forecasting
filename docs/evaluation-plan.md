# 수요예측 평가 계획

## 1. 예측 계약

모든 실험은 target, 예측 발행 시각, horizon, 집계 간격, 평가 대여소 집합을 먼저 기록한다. 첫 target은 대여소별 시간당 대여 시작 건수다.

## 2. 시간 분할

- 오래된 연속 구간을 Train, 다음 연속 구간을 Validation, 가장 최근 연속 구간을 잠긴 Test로 둔다.
- 모델·특징·hyperparameter 선택은 Train과 Validation에서만 수행한다.
- 여러 시작점의 안정성은 expanding-window 또는 rolling-origin으로 확인한다.
- 신규 대여소는 cold-start로 따로 보고하거나 최소 이력 기준을 사전에 고정한다.

정확한 날짜는 데이터 제공 기간과 정책·대여소 변화 시점을 확인한 뒤 확정한다.

## 3. Baseline

- 지속성: 직전 시간 수요
- 학습기간 평균: 대여소별 또는 사전 정의한 그룹 평균
- 전일 동일 시간 seasonal naive
- 전주 동일 시간 seasonal naive

Random Forest, LightGBM, LSTM은 반드시 동일 평가행에서 이 기준과 비교한다.

## 4. 실험군

| 실험 | 특징 |
|------|------|
| Model A | 시간·요일·계절, 과거 수요 lag와 shift된 rolling 통계 |
| Model B | Model A + 예측 시점 가용 기상 + 공휴일 |
| Model C | Model B + GIS 특징 + 검증된 행사 정보 |

Model C는 행사 go/no-go 기준을 통과하지 못하면 생략하고 그 이유를 기록한다.

## 5. 누수 검사

- lag와 rolling은 현재 target보다 최소 한 시점 이전 값만 포함한다.
- 결측 대치, scaling, encoding, feature selection은 Train에만 fit한다.
- 대여소 평균·순위·군집은 해당 fold의 과거 데이터로만 계산한다.
- Test는 hyperparameter 선택, 조기 종료 기준, 특징 선택에 사용하지 않는다.
- 미래 실측 날씨는 주 평가 입력에서 제외한다.
- 행사 수정·취소·관객수 등 사후 정보는 당시 가용 snapshot이 없으면 제외한다.
- 공간 정보의 기준일이 예측 시점보다 미래인지 확인한다.

## 6. Metrics와 slice

- MAE: 주 지표. 평균적인 시간당 자전거 대수 오차를 설명한다.
- RMSE: 피크의 큰 오차에 민감한 보조 지표다.
- 전체 micro, 대여소별 macro, 행정구역, 시간대, 요일, 수요 분위별로 보고한다.
- 행사일/비행사일은 표본 수와 함께 보고하고 희소한 행사 표본에서 강한 결론을 내리지 않는다.
- 학습·검증·테스트 성능 차이와 학습 곡선으로 과적합을 분석한다.

## 7. 모델별 추가 검사

- Random Forest/LightGBM: Train과 Validation 차이, tree 복잡도, feature importance 안정성.
- LSTM: seed 반복, 시퀀스 경계, early stopping, 학습 곡선, 추론비용.
- 모든 모델: 동일 horizon, 평가행, target 변환, inverse transform을 사용한다.

## 8. 재현성 기록

- 데이터 snapshot/checksum과 스키마 버전
- 분할 날짜, 예측 horizon, 제외 대여소
- 코드 버전, Python·패키지 버전, seed
- 특징 목록과 availability timestamp 정책
- 모델 설정, 학습 종료 시점, artifact ID
- 실패한 실험과 변경 사유

## 9. 보고 규칙

- 베이스라인 대비 절대·상대 개선폭을 함께 제시한다.
- 탐색 결과, oracle 상한, 운영 가능한 결과를 구분한다.
- 한 번의 split이나 전체 평균만으로 모델 우월성을 주장하지 않는다.
- forecast-validation 스킬의 검증 기록이 완성된 실험만 최종 비교표에 포함한다.
