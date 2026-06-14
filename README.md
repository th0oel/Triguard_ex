# TriGuard AI

**Regional Military Manpower Risk Score Analysis & Early Warning System**  
*감염병·병역자원 데이터 기반 지역별 병력운용 조기경보 시스템*

2026 병무청·방위사업청·질병관리청 합동 공공데이터·AI 활용 경진대회 출품작

TriGuard AI는 병무청·질병관리청·방위사업청의 공공데이터를 결합하여 14개 지방병무청 권역별 병력운용 위험도를 실시간으로 산출하는 Streamlit 대시보드입니다. 인력 가용성·감염병 확산·군수물자 조달이라는 세 축을 하나의 통합 Risk Score로 정량화하며, 위험 등급별 rule-based 대응 가이드를 함께 제공합니다. 전 지표는 0–100 범위로 정규화되고, 0 나누기 방지 및 지역명 fuzzy matching이 자동 처리됩니다.

---

## 1. Project Structure

```
triguard-ai/
├── app.py               # Streamlit main app — 파일 업로드·파싱·계산·렌더링 오케스트레이션
├── requirements.txt
└── modules/
    ├── preprocess.py    # 인코딩 자동감지, 지역명 표준화, CSV 파싱 함수군
    ├── risk_engine.py   # Risk Score 계산 엔진 + 가중치 상수 (WEIGHTS_*)
    ├── ml_engine.py     # RandomForest 기반 AI 위험 예측, 이상 탐지, 추세 분석
    └── visualize.py     # Streamlit 시각화, KPI 카드, 지도, AI 결과, 대응 가이드 렌더링
```

---

## 2. Getting Started

```bash
pip install -r requirements.txt
streamlit run app.py          # → http://localhost:8501
```

**Google Colab**

```python
!pip install streamlit pyngrok chardet -q
from pyngrok import ngrok
!streamlit run app.py &
print(ngrok.connect(8501))
```

---

## 3. Input Data

| Institution | File | Purpose |
|-------------|------|---------|
| 병무청 | `병역판정검사_현황.csv` | 인력 Risk 핵심 |
| 병무청 | `현역병_입영현황.csv` | 입영 차질률 |
| 병무청 | `병역면제자_관리현황.csv` | 면제율 보조 |
| 질병관리청 | `지역별_감염병_발생현황.csv` | 지방청별 발생률 |
| 질병관리청 | `질병별_감염병_발생현황.csv` | 등급 가중합 |
| 질병관리청 | `인플루엔자_표본감시.csv` | 유행 강도 지수 |
| 질병관리청 | `급성호흡기감염증.csv` | ARI 트렌드 지수 |
| 방위사업청 | `국내조달_계약정보.csv` | 물자 Risk 핵심 |
| 방위사업청 | `국외조달_계약정보.csv` | 국외 의존도 |
| 방위사업청 | `국내조달_입찰참여업체정보.csv` | HHI·공급업체다양성지수 |
| 무역안보관리원 | `전략물자_품목키워드.csv` | 전략물자 비율 |
| 행정안전부 | `지역별_연령별_주민등록_인구현황.csv` | 병역자원비율 보조 |

파일 미업로드 시 시뮬레이션 모드(랜덤 시드 기반 더미 데이터)로 전환됩니다.

---

## 4. Scoring Model

모든 점수는 `clip(0, 100)` 처리됩니다.

### 4-1. 인력 Risk Score

```
인력Risk = 0.30 × 입영감소율
         + 0.25 × 검사감소율
         + 0.20 × 면제율
         + 0.10 × 재검율
         + 0.15 × 차질률
```

| 지표 | 산출식 |
|------|--------|
| 입영인원 감소율 | `(전년 현역 − 당해 현역) / 전년 현역 × 100` |
| 병역판정검사 감소율 | `(전년 처분인원 − 당해 처분인원) / 전년 처분인원 × 100` |
| 병역면제율 | `병역면제자 / 처분인원 × 100` |
| 재신체검사율 | `재신체검사 / 처분인원 × 100` |
| 입영 차질률 | `(행방불명 + 기피) / 입영실통지 × 100` |

### 4-2. 감염병 DC (Disruption Coefficient)

**시도 → 지방청 매핑** (`preprocess.py: SIDO_TO_JIBANG_MAP`)

| 유형 | 예시 |
|------|------|
| 1:1 | 서울→서울, 충북→충북 |
| N:1 (평균) | 부산+울산 → 부산울산 |
| 1:N (동일값 복사) | 경기 → 경인, 경기북부 |
| 세종 포함 | 세종+대전+충남 → 대전충남 |

```
DC_지방청 = 0.35 × 발생률지수_지방청
           + 0.30 × 질병등급가중합지수
           + 0.20 × 인플루엔자강도지수
           + 0.15 × ARI트렌드지수
```

| 구성요소 | 산출식 | 범위 |
|----------|--------|------|
| 발생률지수 (지방청별) | `총발생률 / max(전국) × 100` | 지방청별 상이 |
| 질병등급 가중합지수 | `ln(1 + 가중발생합계) × 5` (1급×3, 2급×2, 3급×1) | 전국 단일 |
| 인플루엔자 강도지수 | `최근 절기 최대 의사환자분율 × 2.5` (미제공 시 30.0) | 전국 단일 |
| ARI 트렌드지수 | `50 + 전년대비증감률 × 100` (미제공 시 30.0) | 전국 단일 |

발생률지수만 지방청별로 다르게 산출되며, 나머지 세 지표는 전국 단일값입니다.

### 4-3. 물자 Risk Score  *(전국 단일값 → 14개 지방청 동일 적용)*

```
물자Risk = 0.25 × 국내감소율지수
         + 0.25 × 국외의존도
         + 0.20 × 집중도
         + 0.15 × 수의의존도
         + 0.15 × 전략물자비율
```

| 지표 | 산출식 |
|------|--------|
| 국내조달 감소율지수 | `100 − (국내건수 / 전체건수 × 100)` |
| 국외조달 의존도 | `국외건수 / 전체건수 × 100` |
| 공급업체 집중도 | `상위 5개 업체 건수 / 국내 총건수 × 100` |
| 수의계약 의존도 | `수의계약건수 / 국내 총건수 × 100` |
| 전략물자 비율지수 | `전략물자 품목수 / 국내 총건수 × 1000` |
| HHI (허핀달-허쉬만 지수) | `Σ(업체 점유율²)` — 0(완전경쟁) ~ 1(독점) |
| 공급업체다양성지수 | `(1 − HHI) / (1 − 1/N) × 100` — 0~100, 높을수록 안전 |

### 4-4. 통합 Risk Score

```
통합Risk_지방청 = 0.40 × 인력Risk
               + 0.40 × DC_지방청
               + 0.20 × 물자Risk
```

### 4-5. Risk Grade

| Grade | 조건 | 조치 |
|-------|------|------|
| 위험 (Red) | 통합Risk ≥ 60 | 즉각 대응 |
| 주의 (Yellow) | 35 ≤ 통합Risk < 60 | 모니터링 강화 |
| 정상 (Green) | 통합Risk < 35 | 현 수준 유지 |

---

## 5. Key Code

### preprocess.py

- `normalize_region_sido / normalize_region_jibang` — 원본 지역명을 표준화. 공백·괄호 제거 → 사전 매핑 → startswith → fuzzy matching (cutoff 0.6–0.75) 순서로 fallback.
- `aggregate_disease_by_jibang(regional_df)` — `SIDO_TO_JIBANG_MAP`을 사용해 17개 시도를 14개 지방청 단위로 집계. N:1 매핑은 평균, 1:N 매핑은 동일값 복사.
- `parse_infectious_disease_regional / _national` — 지역별 총발생률 합산(광역 단위 추출) 및 1·2·3급 가중합(가중치 3·2·1) 계산.
- `parse_population` — 행정안전부 인구 데이터 파싱. 병역자원비율 = `20대 남성 / 총인구 × 100`, 지방청 단위 집계.
- `parse_dapa_bidders` — 입찰참여업체 정보에서 HHI(허핀달-허쉬만 지수, 0–1) 및 공급업체다양성지수(0–100) 산출.
- `safe_divide(num, denom)` — 분모 0·NaN 방지. 전 모듈에서 나눗셈에 일괄 적용.

### risk_engine.py

- `WEIGHTS_*` 상수 — 인력/감염병/물자/통합 각 가중치 딕셔너리. 합계가 1.0을 벗어나면 앱 시작 시 경고 표시.
- `calc_disease_dc(..., jibang_disease_df)` — 지방청별 발생률지수를 계산 후 DC를 산출, `jibang_dc_df[지방청, 발생률지수, 감염병DC]`를 반환.
- `calc_integrated_risk(..., jibang_dc_df)` — `manpower_df`와 `jibang_dc_df`를 LEFT JOIN. 매핑 실패한 지방청은 전국 대표값으로 fallback하고 경고를 반환.

### ml_engine.py

- `train_risk_model(result_df)` — 통합 Risk Score 기반 RandomForest 분류 모델 학습. 교차검증 점수(cv_score) 반환.
- `predict_risk(model, scaler, result_df)` — 학습된 모델로 위험등급 예측 및 클래스별 확률(정상·주의·위험) 반환.
- `cross_agency_correlation(manpower_df, jibang_dc_df)` — 인력 Risk와 감염병 DC 간 상관계수 산출 및 복합 위험 권역 탐지.
- `detect_anomaly_regions(result_df)` — Z-score ≥ 1.5 기준으로 통합 Risk 이상 권역 탐지.
- `forecast_manpower(exam_df, forecast_years)` — 처분인원 연도별 추세 선형 회귀로 향후 N년 예측.

### visualize.py

- `render_kpi_cards` — 위험/주의/정상/전체 권역 수를 4개 metric 카드로 표시.
- `render_map(result_df, score_col)` — 지방청별 Risk Score를 지도 위에 시각화.
- `apply_risk_style` — 점수(≥60 빨강, ≥35 노랑, 미만 초록) 및 등급 셀에 배경색 적용.
- `render_response_guide` — 위험·주의 권역에 대해 주요 위험 요인(인력·감염병·물자 중 최고 점수)을 판별 후 rule-based 권고 조치 출력.
- `render_ml_prediction` — Rule-based 등급과 ML 예측 등급 비교 테이블 및 불일치 권역 강조.
- `render_feature_importance` — RandomForest 피처 중요도 바 차트 렌더링.
- `render_cross_agency_scatter` — 인력 Risk vs 감염병 DC 산점도 및 상관계수 표시.
- `render_manpower_trend` — 처분인원 연도별 추세 및 미래 예측 라인 차트.
- `render_anomaly_table` — Z-score 기반 이상 탐지 결과 테이블 렌더링.

---

## 6. Team

| Role | Name | Affiliation |
|------|------|-------------|
| 팀장 | 최희찬 | — |
| 팀원 | 김기호 | 한국공학대학교 |
| 팀원 | 김태희 | 충북대학교 |
