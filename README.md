# 정수장 AI 지능화 운영 시스템

광주광역시 **덕남·용연 정수장**을 대상으로 한 AI 기반 지능화 운영 플랫폼입니다.  
응집제 주입, 혼화·응집, 소독 공정에 대해 AI 추천 → 운영자 승인 → 단계적 자동화 구조로 운영합니다.

---

## 핵심 설계 원칙

1. **정수장 완전 분리**: 덕남·용연은 공정 구성, 약품 투입 위치, 혼화 방식, 계측 장비가 달라 모델·설정·데이터를 완전히 분리합니다.
2. **군집별 모델 분기**: 유입수 수질 특성을 군집분석으로 분류한 뒤 해당 군집 전용 AI 모델을 적용합니다.
3. **단계적 전환**: AI분석 → AI추천 → AI운영 순서로 점진적으로 자동화 수준을 높입니다.

---

## 전체 처리 흐름

```
실시간 데이터 수집 (정수장별 분리)
→ 데이터 전처리 / 검증
→ 유입수 군집 분류 (정수장별 군집분류기)
→ 군집별 AI 모델 선택
→ 공정별 AI 예측 (약품 / 혼화응집 / 소독 / 수질예측)
→ 안전제약 기반 권고값 산정
→ XAI 근거 생성
→ API / HMI 연계
→ 운영자 승인 또는 제한적 자동적용
→ 결과 피드백 및 MLOps 재학습
```

---

## 디렉토리 구조

```
EPSEnE_water_treatment/
│
├── app/                            # FastAPI 기반 AI API 서버
│   ├── api/                        # 라우터 (엔드포인트 정의)
│   ├── services/                   # 비즈니스 로직 (군집분류, 예측, 권고)
│   ├── schemas/                    # Pydantic 입출력 스키마
│   ├── core/                       # 설정, 의존성, 미들웨어
│   └── main.py                     # FastAPI 앱 진입점
│
├── data_pipeline/                  # 데이터 수집 및 가공 파이프라인
│   ├── collector/
│   │   ├── deoknam/                # 덕남 HMI/SCADA 태그 수집
│   │   └── yongyeon/               # 용연 HMI/SCADA 태그 수집
│   ├── preprocessing/              # 결측·이상치 처리, 칼만필터 기반 센서 스무싱
│   ├── feature_engineering/        # 래그·롤링 피처, HRT 반영, 피처 스토어
│   └── validation/                 # 데이터 품질 검사
│
├── models/                         # AI 모델 코드 (학습·추론 스크립트)
│   ├── cluster/                    # 유입수 군집분류기
│   │   ├── deoknam_classifier/     # 덕남 군집분류기 (K-Means + GMM)
│   │   └── yongyeon_classifier/    # 용연 군집분류기 (K-Means + GMM)
│   │
│   ├── coagulant/                  # 응집제 주입률 예측 모델
│   │   ├── deoknam/
│   │   │   ├── cluster_c1/         # C1 평상시 저탁도
│   │   │   ├── cluster_c2/         # C2 중탁도
│   │   │   ├── cluster_c3/         # C3 강우기 고탁도
│   │   │   ├── cluster_c4/         # C4 저수온기
│   │   │   └── cluster_c5/         # C5 조류번성기
│   │   └── yongyeon/
│   │       ├── cluster_c1/
│   │       ├── cluster_c2/
│   │       ├── cluster_c3/
│   │       ├── cluster_c4/
│   │       └── cluster_c5/
│   │
│   ├── mixing/                     # 혼화·응집 G값/RPM 권고 모델
│   │   ├── deoknam/                # 기계식 급속혼화 (Letterman 산식 기반)
│   │   └── yongyeon/               # 복수 혼화방식 + 플록분석장치 피드백
│   │
│   ├── chlorine/                   # 소독 공정 염소 주입률 예측 모델
│   │   ├── deoknam/
│   │   └── yongyeon/
│   │
│   ├── process_prediction/         # 침전·여과·정수 수질 예측 모델
│   │   ├── deoknam/
│   │   └── yongyeon/
│   │
│   └── anomaly_detection/          # 이상탐지 모델 (군집 전이 탐지 포함)
│       ├── deoknam/
│       └── yongyeon/
│
├── simulation/                     # What-if 시뮬레이션 엔진
│   ├── what_if_engine.py           # 운영조건 변경 시뮬레이션
│   └── optimization.py             # Bayesian Optimization 기반 최적화
│
├── mlops/                          # MLOps 파이프라인
│   ├── train.py                    # 군집별 모델 학습
│   ├── evaluate.py                 # 군집별 성능 평가 (R², MAE, RMSE)
│   ├── registry.py                 # MLflow 모델 레지스트리 연동
│   ├── drift_monitor.py            # 데이터·성능 드리프트 감지
│   └── rollback.py                 # 이전 모델 롤백
│
├── configs/                        # 정수장별 설정 파일
│   ├── deoknam.yaml                # 덕남 태그 매핑, 안전제약, 모델 파라미터
│   ├── yongyeon.yaml               # 용연 태그 매핑, 안전제약, 모델 파라미터
│   └── common.yaml                 # 공통 설정 (DB, API, 재학습 스케줄)
│
├── docker/                         # 컨테이너 배포 설정
│   ├── Dockerfile.api              # FastAPI API 서버 이미지
│   ├── Dockerfile.worker           # 모델 추론 워커 이미지
│   └── docker-compose.yml          # 전체 서비스 구성
│
├── ml/                             # 모델 학습 작업 공간 (정수장·군집별 분리)
│   ├── 01_clustering/              # 유입수 군집분류기 (K-Means + GMM)
│   │   ├── deoknam/                # 덕남 전용 군집분류기
│   │   └── yongyeon/               # 용연 전용 군집분류기
│   │
│   ├── 02_coagulant/               # 응집제 주입률 예측 (XGBoost / LightGBM)
│   │   ├── deoknam/                # 침전수 탁도 피드백 기반 역산
│   │   │   ├── cluster_c1/         # C1 평상시 저탁도
│   │   │   ├── cluster_c2/         # C2 중탁도
│   │   │   ├── cluster_c3/         # C3 강우기 고탁도
│   │   │   ├── cluster_c4/         # C4 저수온기
│   │   │   └── cluster_c5/         # C5 조류번성기
│   │   └── yongyeon/               # Jar-Test 기반 직접 예측
│   │       ├── cluster_c1/
│   │       ├── cluster_c2/
│   │       ├── cluster_c3/
│   │       ├── cluster_c4/
│   │       └── cluster_c5/
│   │
│   ├── 03_mixing/                  # 혼화·응집 G값/RPM (Letterman 산식 + ML 보정)
│   │   ├── deoknam/                # 기계식 급속혼화 단일 방식
│   │   │   ├── cluster_c1/
│   │   │   ├── cluster_c2/
│   │   │   ├── cluster_c3/
│   │   │   ├── cluster_c4/
│   │   │   └── cluster_c5/
│   │   └── yongyeon/               # 복수 혼화방식 + 플록분석장치 피드백
│   │       ├── cluster_c1/
│   │       ├── cluster_c2/
│   │       ├── cluster_c3/
│   │       ├── cluster_c4/
│   │       └── cluster_c5/
│   │
│   ├── 04_chlorine/                # 소독 염소 주입률 (XGBoost / LSTM)
│   │   ├── deoknam/
│   │   │   ├── cluster_c1/
│   │   │   ├── cluster_c2/
│   │   │   ├── cluster_c3/
│   │   │   ├── cluster_c4/
│   │   │   └── cluster_c5/
│   │   └── yongyeon/
│   │       ├── cluster_c1/
│   │       ├── cluster_c2/
│   │       ├── cluster_c3/
│   │       ├── cluster_c4/
│   │       └── cluster_c5/
│   │
│   ├── 05_process_prediction/      # 침전·여과·정수 수질 예측 (XGBoost / LSTM)
│   │   ├── deoknam/
│   │   │   ├── cluster_c1/
│   │   │   ├── cluster_c2/
│   │   │   ├── cluster_c3/
│   │   │   ├── cluster_c4/
│   │   │   └── cluster_c5/
│   │   └── yongyeon/
│   │       ├── cluster_c1/
│   │       ├── cluster_c2/
│   │       ├── cluster_c3/
│   │       ├── cluster_c4/
│   │       └── cluster_c5/
│   │
│   ├── 06_anomaly_detection/       # 이상탐지 (군집 전이 탐지 포함)
│   │   ├── deoknam/
│   │   └── yongyeon/
│   │
│   └── utils/                      # 공통 유틸 (데이터 로더, 전처리, 평가 지표)
│
├── dataset/                        # 학습용 샘플 데이터셋
│   ├── 덕남_소독공정_1분.parquet
│   ├── 덕남_응집제공정_1분.parquet
│   ├── 용연_소독공정_1분.parquet
│   └── 용연_응집제공정_1분.parquet
│
├── docs/                           # 설계 문서
│   ├── 정수장_AI_지능화_운영시스템_상세설계안.md
│   └── 광주광역시 2단계_AI 기술부분.pdf
│
├── static/                         # UI 프로토타입
│   └── AI정수장 화면설계.html
│
└── tests/                          # 테스트
    ├── unit/                       # 단위 테스트 (모델·서비스·유틸)
    ├── integration/                # 통합 테스트 (파이프라인 전체)
    └── api/                        # API 엔드포인트 테스트
```

---

## 정수장별 공정 비교

| 항목 | 덕남정수장 | 용연정수장 |
|---|---|---|
| 혼화 방식 | 기계식 급속혼화 (단일) | 복수 혼화방식 선택 가능 |
| 응집플록분석장치 | 없음 | 있음 |
| Jar-Test 기록 | 미흡 | 체계화 |
| 자동화 적합성 | 중간 | 높음 |
| 초기 모델 전략 | 침전수 탁도 피드백 기반 역산 | Jar-Test 기반 직접 예측 |

---

## 유입수 군집 정의

| 군집 | 명칭 | 주요 특성 | 대응 전략 |
|---|---|---|---|
| C1 | 평상시 저탁도 | 탁도 < 5 NTU, 정상 pH·수온 | 표준 응집 운전 |
| C2 | 중탁도 | 탁도 5~30 NTU | 응집제 증량, G값 상향 |
| C3 | 강우기 고탁도 | 탁도 > 30 NTU, 급격한 변화율 | 고탁도 전용 모델, 회귀식 보정 |
| C4 | 저수온기 | 수온 < 10°C | 점도 보정, G값 상향 |
| C5 | 조류번성기 | TOC 높음, Chl-a 상승 | 분말활성탄 병행, 전처리 강화 |

군집 수와 경계는 실제 데이터 분석(Elbow method, Silhouette Score) 후 정수장별로 독립 확정합니다.

---

## 주요 AI 모델

| 모델 | 알고리즘 | 입력 | 출력 |
|---|---|---|---|
| 군집분류기 | K-Means + GMM | 원수 탁도·pH·수온·EC·알칼리도·TOC | 군집 레이블 (C1~C5) |
| 응집제 주입률 | XGBoost / LightGBM | 원수 수질, 유량, 군집 레이블 | 응집제 권고 주입률 (ppm) |
| G값/RPM | Letterman 산식 + ML 보정 | 수온, 점도, 응집제 주입률, 군집 | 목표 G값, 권고 RPM |
| 소독 염소 | XGBoost / LSTM | 수온·TOC·Mn, 유량, HRT, 군집 | 전/중/후 염소 권고값 |
| 수질 예측 | XGBoost / LSTM | 운영조건 + 약품 주입률 | 침전수·여과수·잔류염소 예측 |
| 이상탐지 | Isolation Forest + Rule | 센서값, 운영이력, 군집 경계 | 이상 플래그, 군집 전이 경고 |

---

## 핵심 API

| API | Method | 기능 |
|---|---|---|
| `/api/v1/cluster/classify` | POST | 유입수 군집 분류 |
| `/api/v1/models/predict/coagulant` | POST | 응집제 주입률 예측 |
| `/api/v1/models/predict/mixing` | POST | G값/RPM 예측 |
| `/api/v1/models/predict/chlorine` | POST | 염소 주입률 예측 |
| `/api/v1/recommendations/latest` | GET | 최신 AI 권고값 조회 |
| `/api/v1/recommendations/approve` | POST | 운영자 승인 |
| `/api/v1/simulation/what-if` | POST | 운영조건 변경 시뮬레이션 |
| `/api/v1/anomaly/latest` | GET | 최신 이상탐지 결과 |
| `/api/v1/xai/{prediction_id}` | GET | XAI 설명 조회 |
| `/api/v1/mlops/retrain` | POST | 모델 재학습 실행 |
| `/api/v1/mlops/rollback` | POST | 이전 모델 롤백 |

---

## 인프라 구성

| 컨테이너 | 역할 |
|---|---|
| `ai-api-server` | FastAPI 추론 API |
| `data-collector` | HMI/SCADA 실시간 데이터 수집 |
| `preprocessor` | 전처리 + 칼만필터 센서 스무싱 |
| `cluster-classifier` | 유입수 군집 분류 (정수장별) |
| `model-worker` | 군집별 모델 추론 |
| `simulation-worker` | What-if 시뮬레이션 |
| `mlops-server` | MLflow 모델 관리 |
| `scheduler` | 재학습 / 배치 작업 |
| `postgres` | 운영 DB |
| `redis` | 캐시 / 큐 |
| `nginx` | API Gateway |

---

## 모델 버전 규칙

```
{plant_id}_{process}_{cluster}_{model_type}_{yyyymmdd}_{version}

예시:
deoknam_coagulant_c1_xgb_20260517_v1.0.0
yongyeon_chlorine_c2_lstm_20260517_v1.0.0
deoknam_cluster_classifier_20260517_v1.0.0
```

---

## 안전 제약 (Safety Guardrail)

- 주입률 상한/하한: 과거 운영 이력 q1~q99 또는 감독원 승인값
- 1회 변경폭 제한: 최근 운영 이력 기반 산정
- 군집 전이 감지 시: 자동적용 즉시 중단, 운영자 알림
- 미학습 수질 범위: 자동적용 금지, 추천만 허용
- 센서 이상 감지 시: AI 권고 중지

---

## 개발 단계

| 단계 | 내용 |
|---|---|
| 1단계 | 요구사항 분석 · 태그 정의 · 운영자 인터뷰 |
| 2단계 | 데이터 마트 구축 · EDA · 군집분석 |
| 3단계 | 군집별 모델 개발 (약품 / 혼화 / 소독 / 수질예측 / 이상탐지) |
| 4단계 | API 서버 개발 · HMI 연계 · XAI 표출 |
| 5단계 | 시운전 · MLOps 체계 · 24시간 무중단 테스트 |

---

## 참고 문서

- [정수장 AI 지능화 운영시스템 상세설계안](docs/정수장_AI_지능화_운영시스템_상세설계안.md)
- [광주광역시 2단계 AI 기술부분](docs/광주광역시%202단계_AI%20기술부분.pdf)
- [AI 운영화면 설계 프로토타입](static/AI정수장%20화면설계.html)
