# Spatial Demand Forecasting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 서울시 따릉이의 대여소별 시간당 대여건수를 예측하고 외부 영향요인을 검증하는 재현 가능한 GIS·ML 포트폴리오를 단계적으로 구축한다.

**Architecture:** 공식 출처를 검증한 원천 데이터를 Pandas/GeoPandas로 정제하고 PostgreSQL/PostGIS의 raw, staging, mart 계층에 적재한다. 같은 시간 분할에서 베이스라인, Random Forest, LightGBM, LSTM과 외부 변수 조합을 비교한 뒤, 사전 계산한 예측을 FastAPI와 Streamlit으로 제공한다.

**Tech Stack:** Python, Pandas, NumPy, PostgreSQL/PostGIS, GeoPandas, QGIS, scikit-learn, LightGBM, PyTorch, FastAPI, Streamlit, pytest, Docker

**Spec:** `docs/superpowers/specs/2026-10-30-spatial-demand-forecasting-design.md`

## Global Constraints

- 개발 순서는 SQL/GIS -> 머신러닝 -> 딥러닝 -> API/대시보드다.
- 예측 대상은 우선 `station_id × hour`의 대여건수다.
- 데이터셋, 실제 컬럼명, 기간, 라이선스는 공식 출처 확인 전 확정하지 않는다.
- 예측 시점에 알 수 없는 실제 미래 날씨와 사후 확정 행사정보를 입력으로 사용하지 않는다.
- 테스트 구간은 모델 선택에 사용하지 않으며 모든 변환은 학습 구간에만 fit한다.
- 각 기능은 실패 테스트 -> 최소 구현 -> 전체 테스트 순서로 개발한다.

## Review Focus

- 대여소 ID 변경·이전·폐쇄가 시점에 맞는 위치와 결합되는지 확인한다.
- DST를 포함한 시간대 정규화와 시간 집계 경계가 일관적인지 확인한다.
- lag와 rolling 특징이 현재 또는 미래 target을 포함하지 않는지 확인한다.
- 실제 예측 시점에 이용할 수 없는 날씨·행사 데이터가 섞이지 않는지 확인한다.
- 신규 대여소와 행사 표본 부족이 전체 평균 성능에 가려지지 않는지 확인한다.

---

## 파일 구조와 책임

- `src/spatial_demand/config.py`: 경로, 시간대, 예측 horizon, 분할 경계 설정.
- `src/spatial_demand/data/contracts.py`: 검증된 원천 데이터 계약.
- `src/spatial_demand/data/ingest.py`: 원천 파일 읽기와 raw 적재 입력 생성.
- `src/spatial_demand/features/temporal.py`: 시간·lag·rolling 특징.
- `src/spatial_demand/features/spatial.py`: 거리·행정구역·주변 시설 특징.
- `src/spatial_demand/evaluation/splits.py`: 시간순 분할과 rolling-origin fold.
- `src/spatial_demand/evaluation/metrics.py`: MAE/RMSE 및 그룹별 성능.
- `src/spatial_demand/models/baselines.py`: 지속성·평균·계절 naive.
- `src/spatial_demand/models/tree.py`: Random Forest와 LightGBM.
- `src/spatial_demand/models/lstm.py`: PyTorch Dataset과 LSTM.
- `src/spatial_demand/api/main.py`: 읽기 전용 예측 API.
- `src/spatial_demand/dashboard/app.py`: Streamlit 지도·비교 화면.
- `sql/ddl`: raw, staging, mart 테이블과 인덱스.
- `sql/queries`: 품질 검사, 시간 집계, 공간 조인.
- `tests`: 위 모듈 구조를 따르는 단위·통합 테스트.

### Task 1: 공식 데이터 출처 검증과 계약 확정

**Files:**
- Modify: `docs/data-source-research.md`
- Create: `docs/data-catalog.md`
- Create: `src/spatial_demand/data/contracts.py`
- Test: `tests/data/test_contracts.py`

**Interfaces:**
- Consumes: 공식 데이터 사전, 샘플 파일, 라이선스와 기준일.
- Produces: 원천별 필수 의미 필드, 실제 컬럼 매핑, 시간대, CRS, 결합 키 계약.

- [ ] **Step 1:** 공식 출처 URL, 조회일, 제공 기간, 라이선스, 샘플 체크섬을 카탈로그에 기록한다.
- [ ] **Step 2:** 필수 의미 필드가 없는 샘플을 거부하는 실패 테스트를 작성하고 실행한다.
- [ ] **Step 3:** 검증된 실제 컬럼만 포함한 데이터 계약을 최소 구현한다.
- [ ] **Step 4:** `pytest tests/data/test_contracts.py -v`와 전체 `pytest`를 실행한다.
- [ ] **Step 5:** 데이터 출처와 계약을 커밋한다.

### Task 2: SQL/PostGIS 적재와 시공간 마트

**Files:**
- Create: `sql/ddl/001_extensions_and_schemas.sql`
- Create: `sql/ddl/002_core_tables.sql`
- Create: `sql/queries/build_station_hour.sql`
- Create: `sql/queries/assign_admin_area.sql`
- Create: `src/spatial_demand/data/ingest.py`
- Test: `tests/integration/test_spatial_mart.py`

**Interfaces:**
- Consumes: Task 1 데이터 계약과 정제된 원천 행.
- Produces: `mart.station_hour`, 시점 유효 대여소 geometry, 행정구역 매핑.

- [ ] **Step 1:** 중복, 누락 시간, 잘못된 좌표, 경계선 점을 포함한 실패 통합 테스트를 작성한다.
- [ ] **Step 2:** raw/staging/mart DDL과 공간 인덱스를 구현한다.
- [ ] **Step 3:** 시간 집계와 명시적 경계 포함 규칙의 공간 조인을 구현한다.
- [ ] **Step 4:** 통합 테스트와 전체 `pytest`를 실행하고 QGIS로 샘플 geometry를 확인한다.
- [ ] **Step 5:** SQL/GIS 마트를 커밋한다.

### Task 3: 시간 분할, 누수 방지 특징, 베이스라인

**Files:**
- Create: `src/spatial_demand/evaluation/splits.py`
- Create: `src/spatial_demand/features/temporal.py`
- Create: `src/spatial_demand/models/baselines.py`
- Test: `tests/evaluation/test_splits.py`
- Test: `tests/features/test_temporal.py`
- Test: `tests/models/test_baselines.py`

**Interfaces:**
- Produces: chronological split, rolling-origin fold, lag/shift 특징, persistence·mean·seasonal-naive 예측.

- [ ] **Step 1:** 무작위 분할과 미래 target을 포함한 rolling 특징을 거부하는 실패 테스트를 작성한다.
- [ ] **Step 2:** 시간순 분할과 shift 이후 rolling 특징을 구현한다.
- [ ] **Step 3:** 지속성, 학습 구간 평균, 전일·전주 동일 시간 베이스라인을 구현한다.
- [ ] **Step 4:** 대상 테스트와 전체 `pytest`를 실행한다.
- [ ] **Step 5:** 분할·특징·베이스라인을 커밋한다.

### Task 4: Random Forest와 LightGBM

**Files:**
- Create: `src/spatial_demand/models/tree.py`
- Create: `src/spatial_demand/evaluation/metrics.py`
- Test: `tests/models/test_tree.py`
- Test: `tests/evaluation/test_metrics.py`

**Interfaces:**
- Consumes: Task 3의 분할·특징·베이스라인.
- Produces: Model A/B 트리 모델 결과와 MAE/RMSE·지역별 성능 표.

- [ ] **Step 1:** fit 범위와 그룹별 MAE/RMSE를 검증하는 실패 테스트를 작성한다.
- [ ] **Step 2:** Random Forest와 LightGBM 학습 인터페이스를 구현한다.
- [ ] **Step 3:** Model A와 B를 동일 fold에서 평가하고 베이스라인 대비 개선폭을 기록한다.
- [ ] **Step 4:** 대상 테스트와 전체 `pytest`를 실행한다.
- [ ] **Step 5:** 머신러닝 비교를 커밋한다.

### Task 5: 공간·행사 특징과 Model C

**Files:**
- Create: `src/spatial_demand/features/spatial.py`
- Create: `src/spatial_demand/features/events.py`
- Test: `tests/features/test_spatial.py`
- Test: `tests/features/test_events.py`

**Interfaces:**
- Consumes: 검증된 지하철역·주요시설·행사장 geometry와 행사 유효시간.
- Produces: 최근접 거리, 반경 내 시설 수, 행사 영향 플래그와 Model C 특징.

- [ ] **Step 1:** CRS 불일치, 행사 취소, 예측 후 확정 이벤트를 거부하는 실패 테스트를 작성한다.
- [ ] **Step 2:** 지역 투영 CRS에서 거리·반경 특징을 구현한다.
- [ ] **Step 3:** 검증된 과거 행사만 시공간 범위로 연결하고 미확보 시 기능을 비활성화한다.
- [ ] **Step 4:** Model C를 A/B와 같은 fold에서 평가하고 행사일 표본 수를 함께 기록한다.
- [ ] **Step 5:** 대상 테스트와 전체 `pytest`를 실행한 뒤 공간 특징을 커밋한다.

### Task 6: PyTorch LSTM 비교

**Files:**
- Create: `src/spatial_demand/models/lstm.py`
- Test: `tests/models/test_lstm.py`

**Interfaces:**
- Consumes: Task 3의 시간순 시퀀스와 Model A/B 특징.
- Produces: 고정 seed의 LSTM 예측과 학습·검증 손실 기록.

- [ ] **Step 1:** 시퀀스 경계가 검증·테스트 target을 학습 입력으로 누출하지 않는 실패 테스트를 작성한다.
- [ ] **Step 2:** 최소 Dataset, DataLoader, LSTM 학습·예측 인터페이스를 구현한다.
- [ ] **Step 3:** early stopping과 seed를 기록하고 같은 테스트 구간에서 트리 모델과 비교한다.
- [ ] **Step 4:** 대상 테스트와 전체 `pytest`를 실행한다.
- [ ] **Step 5:** 딥러닝 비교를 커밋한다.

### Task 7: 최종 평가 리포트

**Files:**
- Create: `src/spatial_demand/evaluation/report.py`
- Create: `docs/model-card.md`
- Test: `tests/evaluation/test_report.py`

**Interfaces:**
- Consumes: 모든 모델의 동일 키 예측행.
- Produces: 전체·지역·대여소·행사일·비행사일 MAE/RMSE, 과적합 분석, 모델 카드.

- [ ] **Step 1:** 서로 다른 평가행을 비교하거나 표본 수 없이 행사 성능을 보고하면 실패하는 테스트를 작성한다.
- [ ] **Step 2:** 공통 평가 키와 slice별 지표 리포트를 구현한다.
- [ ] **Step 3:** 학습/검증/테스트 차이, 베이스라인 개선폭, cold-start를 문서화한다.
- [ ] **Step 4:** forecast-validation 스킬 체크리스트와 전체 `pytest`를 실행한다.
- [ ] **Step 5:** 평가 리포트와 모델 카드를 커밋한다.

### Task 8: FastAPI와 Streamlit 서비스

**Files:**
- Create: `src/spatial_demand/api/main.py`
- Create: `src/spatial_demand/dashboard/app.py`
- Test: `tests/api/test_predictions.py`

**Interfaces:**
- Consumes: 버전이 기록된 사전 계산 예측과 대여소 geometry.
- Produces: 상태·대여소·예측 조회 API와 지도 기반 비교 화면.

- [ ] **Step 1:** 잘못된 시간 범위, 없는 대여소, 없는 모델 버전의 API 실패 테스트를 작성한다.
- [ ] **Step 2:** 작은 읽기 전용 FastAPI 인터페이스를 구현한다.
- [ ] **Step 3:** 지역·시간·모델 필터, 예측 지도, 실제값 비교 Streamlit 화면을 구현한다.
- [ ] **Step 4:** API 테스트와 전체 `pytest`를 실행하고 지도 좌표를 수동 확인한다.
- [ ] **Step 5:** 서비스 계층을 커밋한다.

### Task 9: Docker와 공개 포트폴리오 정리

**Files:**
- Create: `Dockerfile`
- Create: `compose.yaml`
- Modify: `README.md`

**Interfaces:**
- Consumes: 검증된 DB, API, 대시보드 실행 명령.
- Produces: 재현 가능한 로컬 실행 환경과 결과 중심 포트폴리오 문서.

- [ ] **Step 1:** 깨끗한 환경에서 필요한 서비스와 환경변수 계약을 문서화한다.
- [ ] **Step 2:** DB/API/대시보드의 최소 컨테이너 구성을 작성한다.
- [ ] **Step 3:** README에 데이터 계보, 지도, 모델 비교, 한계, 재현 절차를 추가한다.
- [ ] **Step 4:** 빌드, 전체 테스트, 문서 링크, 비밀정보·대용량 파일 제외를 검증한다.
- [ ] **Step 5:** 공개 준비 변경을 커밋한다.

## 자체 검토

- 설계 요구사항은 Task 1~9에 연결된다.
- 데이터 검증 전 실제 컬럼명이나 출처를 확정하지 않았다.
- GIS, ML, DL, 서비스가 순차적으로 독립 검증 가능한 산출물을 만든다.
- 모든 모델이 같은 시간 분할과 평가행을 사용한다.
- 행사 데이터가 없을 때 Model C를 억지로 구성하지 않는다.
