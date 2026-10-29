---
name: forecast-validation
description: Use when evaluating, comparing, selecting, or reporting time-dependent demand-forecasting models, especially when weather, holidays, events, geospatial features, or neural networks are involved.
---

# 수요예측 검증

## 핵심 원칙

예측 성능은 **예측 발행 시점에 알 수 있었던 정보만 사용한 시간순 평가**로 입증한다. 필수 기록이 하나라도 없으면 모델 우월성을 확정적으로 보고하지 않는다.

## 사용 시점

- Baseline, Random Forest, LightGBM, LSTM의 성능 비교
- 시간·날씨·공휴일·행사·GIS 특징의 추가 효과 분석
- MAE/RMSE 결과표, 모델 카드, 포트폴리오 성능 주장 검토
- 지역·대여소·행사일별 오차와 과적합 분석

단순 EDA, 지도 작성, 데이터 다운로드 계획에는 사용하지 않는다.

## 필수 검증 기록

평가마다 아래 표를 채운다. 답을 확인할 수 없는 항목은 “미확인”으로 남기고 최종 성능 주장에 사용하지 않는다.

| 항목 | 반드시 기록할 내용 |
|------|--------------------|
| 예측 계약 | target, grain, 발행 시각, horizon, 평가 대여소 |
| 시간 분할 | Train/Validation/Test 경계, rolling-origin 방식, 잠긴 Test 여부 |
| 누수 검사 | lag/rolling 기준시점, 전처리 fit 범위, Test의 모델 선택 사용 여부 |
| 외부 변수 | 각 특징의 availability timestamp; 미래 실측 날씨와 사후 행사정보 제외 여부 |
| Baseline | 지속성, 평균, 전일/전주 seasonal naive 중 적용 모델과 개선폭 |
| 지표 | 동일 평가행의 MAE와 RMSE, 전체 micro와 대여소 macro |
| Slice | 지역·대여소·시간대·행사일/비행사일 성능과 각 표본 수 |
| 과적합 | Train/Validation/Test 차이, 학습 곡선 또는 복잡도, LSTM seed 안정성 |
| 재현성 | 데이터 snapshot, 분할, 특징 목록, seed, 코드·환경·모델 버전 |

## 판정

| 판정 | 조건 |
|------|------|
| 운영 가능한 평가 | 모든 입력이 예측 시점에 가용하고 Test가 모델 선택에 사용되지 않음 |
| Oracle/상한선 | 미래 실측 날씨 등 운영 시 알 수 없는 정보를 의도적으로 사용함 |
| 탐색 결과 | 단일 split, 불완전 baseline, 미확정 데이터로 가설만 확인함 |
| 무효 비교 | 미래 target 정보, Test 튜닝, 서로 다른 평가행 등으로 공정 비교가 불가능함 |

Oracle과 탐색 결과는 별도 표기할 수 있지만 운영 성능이나 최종 모델 우월성으로 보고하지 않는다.

## 외부 변수 확인

- 날씨: 과거 예보 snapshot을 우선한다. 없으면 날씨 제외 또는 Train만으로 만든 기후 평균을 사용한다.
- 행사: 당시 게시된 일정·장소·상태를 재구성할 수 있을 때만 입력으로 쓴다. 최종 관객수, 사후 취소·변경은 입력에서 제외한다.
- GIS: geometry와 시설 정보의 기준일이 예측 시점 이후인지 확인한다.
- 공휴일: 예측 시점에 공표된 임시·대체공휴일 규칙을 기록한다.

## 보고 예시

“LightGBM Model B는 잠긴 2025년 10월 Test에서 seasonal-naive 대비 MAE 8% 개선”처럼 모델, 특징군, 시간 구간, baseline, metric을 함께 쓴다. 같은 표에 RMSE와 지역·대여소 slice, 행사일 표본 수, 재현성 ID를 연결한다.

## 흔한 합리화와 대응

| 합리화 | 대응 |
|--------|------|
| “행사는 이미 과거에 일어났으니 안전하다” | 오늘 수정된 과거 기록은 당시 가용 정보가 아니다. data vintage를 확인한다. |
| “실측 날씨는 나중에 예보로 바꾸면 된다” | 해당 결과는 oracle일 뿐이다. 운영 성능과 분리한다. |
| “LSTM이 LightGBM보다 낮으니 충분하다” | 동일 split·평가행·baseline이 아니면 비교 근거가 아니다. |
| “전체 RMSE가 좋아서 지역별 분석은 불필요하다” | 특정 지역·신규 대여소·행사일 실패가 평균에 가려질 수 있다. |

## 중단 신호

- 무작위 Train/Test 분할
- 같은 holdout에서 튜닝과 최종 보고
- 전체 기간에 fit한 scaling·encoding·평균
- 미래 실측 날씨 또는 사후 확정 행사정보
- baseline 없는 성능 주장
- 표본 수 없는 행사일·지역별 개선 주장

하나라도 발견하면 결과 발표를 멈추고 해당 실험을 탐색·oracle·무효 중 하나로 재분류한다.
