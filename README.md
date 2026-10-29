# Spatial Demand Forecasting

서울시 공공자전거(따릉이)의 **대여소별 시간당 대여건수**를 예측하고, 날씨·공휴일·행사·공간 특성이 성능에 미치는 영향을 검증하는 시공간 수요예측 포트폴리오 프로젝트입니다.

## 현재 상태

현재는 설계 단계입니다. 데이터셋과 실제 컬럼, 제공 기간, 라이선스는 아직 검증하지 않았으며 데이터 다운로드·모델 학습·DB 구축·서비스 구현도 시작하지 않았습니다.

## 개발 목표

- 공식 출처 기반 데이터 계보와 품질 검사를 구축합니다.
- PostGIS와 GeoPandas로 행정구역 결합, 거리·영역 분석 등 실제 공간 연산을 수행합니다.
- 지속성·평균 베이스라인부터 Random Forest, LightGBM, LSTM까지 동일한 시간 분할에서 비교합니다.
- Model A(시간·과거수요), B(+기상·공휴일), C(+GIS·검증된 행사)의 외부 변수 효과를 비교합니다.
- MAE/RMSE뿐 아니라 지역·대여소·행사일별 성능과 과적합을 분석합니다.
- 최종 예측을 FastAPI와 Streamlit 지도에서 설명 가능한 형태로 제공합니다.

## 점진적 기술 스택

| 단계 | 기술 |
|------|------|
| 데이터·SQL | Python, Pandas, NumPy, PostgreSQL |
| GIS | PostGIS, GeoPandas, QGIS, pyproj/Shapely |
| 머신러닝 | scikit-learn, Random Forest, LightGBM |
| 딥러닝 | PyTorch, LSTM |
| 서비스 | FastAPI, Streamlit, 지도 시각화 |
| 품질·재현성 | pytest, Git/GitHub, Docker |

모든 기술을 처음부터 사용하지 않습니다. 각 단계는 독립적으로 검증 가능한 산출물을 만든 뒤 다음 단계로 넘어갑니다.

## 핵심 원칙

- 예측 시점을 먼저 정의하고 그 시점에 알 수 있던 입력만 사용합니다.
- 미래 실측 날씨는 운영 성능 입력으로 사용하지 않습니다.
- 과거 행사 데이터는 시각·장소·취소·변경 이력과 당시 가용성을 검증한 경우에만 사용합니다.
- 시간순 Train/Validation/Test와 rolling-origin 검증을 사용합니다.
- 테스트 구간은 최종 모델 비교 전까지 잠급니다.
- 재고·반납·재배치 최적화는 관련 데이터가 확보된 경우에만 확장합니다.

## 개발 순서

1. 공식 데이터 출처와 스키마 검증
2. SQL/PostGIS 적재 및 시공간 EDA
3. 베이스라인과 누수 없는 특징 생성
4. Random Forest·LightGBM
5. GIS·행사 외부 변수 비교
6. PyTorch LSTM
7. 최종 평가 리포트
8. FastAPI·Streamlit
9. Docker 및 공개 포트폴리오 정리

자세한 내용은 [개발 로드맵](docs/development-roadmap.md), [데이터 출처 조사 계획](docs/data-source-research.md), [평가 계획](docs/evaluation-plan.md), [아키텍처](docs/architecture.md)를 참고하세요.

## 프로젝트 구조

    .
    ├── .agents/skills/forecast-validation/  # 예측 실험 검증 스킬
    ├── docs/                                # 설계·조사·평가·계획 문서
    ├── notebooks/                           # 탐색·설명용 노트북
    ├── qgis/                                # QGIS 프로젝트와 정적 지도
    ├── sql/ddl/                             # DB 스키마
    ├── sql/queries/                         # 품질·집계·공간 쿼리
    ├── src/spatial_demand/                  # 재사용 가능한 Python 코드
    ├── tests/                               # 단위·통합 테스트
    ├── AGENTS.md                            # 전체 개발 규칙
    ├── pyproject.toml                       # 단계별 선택 의존성
    └── task_plan.md                         # 현재 작업 계획

## 문서 상태

이 저장소의 데이터 관련 표와 필드명은 공식 출처 검증 전에는 “필요 의미 필드” 또는 “후보”입니다. 확인되지 않은 URL이나 컬럼을 사실처럼 사용하지 않습니다.
