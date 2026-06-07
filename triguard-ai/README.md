# 🛡️ TriGuard AI

**감염병·병역자원 데이터 기반 지역별 병력운용 Risk Score 분석 및 조기경보 시스템**

2026 병무청·방위사업청·질병관리청 합동 공공데이터·AI 활용 경진대회 출품작

---

## 📁 폴더 구조

```
triguard-ai/
├── app.py                  # Streamlit 메인 앱
├── requirements.txt
├── data/                   # (선택) 로컬 CSV 보관
└── modules/
    ├── preprocess.py       # 인코딩 감지, 지역명 표준화, CSV 파싱
    ├── risk_engine.py      # Risk Score 계산 엔진 + 가중치 상수
    └── visualize.py        # Streamlit 시각화 + 대응 가이드
```

---

## ⚙️ 실행 방법

### 1. 환경 설치

```bash
pip install -r requirements.txt
```

Python 3.10 이상 권장. Google Colab에서도 동작합니다.

### 2. 앱 실행

```bash
streamlit run app.py
```

브라우저에서 `http://localhost:8501` 접속

### 3. Google Colab에서 실행

```python
!pip install streamlit pyngrok chardet -q
from pyngrok import ngrok
!streamlit run app.py &
public_url = ngrok.connect(8501)
print(public_url)
```

---

## 📂 업로드 파일 목록

| 기관 | 파일명 | 용도 |
|------|--------|------|
| 병무청 | `병역판정검사_현황.csv` | 인력 Risk 핵심 |
| 병무청 | `현역병_입영현황.csv` | 입영 차질률 |
| 병무청 | `병역면제자_관리현황.csv` | 면제율 보조 |
| 질병관리청 | `지역별_감염병_발생현황.csv` | 지역 발생률 |
| 질병관리청 | `질병별_감염병_발생현황.csv` | 등급 가중합 |
| 질병관리청 | `인플루엔자_표본감시.csv` | 유행 강도 |
| 질병관리청 | `급성호흡기감염증.csv` | 트렌드 |
| 방위사업청 | `국내조달_계약정보.csv` | 물자 Risk 핵심 |
| 방위사업청 | `국외조달_계약정보.csv` | 국외 의존도 |
| 무역안보관리원 | `전략물자_품목키워드.csv` | 전략물자 비율 |

> 일부 파일만 업로드해도 가능한 범위에서 계산합니다.  
> 파일 미업로드 시 **시뮬레이션 모드**를 사용하세요.

---

## 🏗️ 가중치 구조

### 통합 Risk Score
| 요소 | 가중치 |
|------|--------|
| 인력 Risk | 40% |
| 감염병 DC | 40% |
| 물자 Risk | 20% |

### 위험 등급
| 등급 | 범위 |
|------|------|
| 🔴 위험 | 60점 이상 |
| 🟡 주의 | 35~60점 |
| 🟢 정상 | 35점 미만 |

> 가중치는 `modules/risk_engine.py` 상단 상수에서 수정 가능합니다.

---

## 📌 주의사항

- 모든 Risk Score는 **0~100** 범위로 clip 처리
- 0 나누기 방지를 위한 `safe_divide()` 함수 전면 적용
- 지역명 표준화: 공백·괄호 제거 → 사전 매핑 → startswith → fuzzy matching 순서
- RAG 대응 가이드: **현재 MVP는 rule-based**. 향후 RAG 기반으로 확장 예정

---

## 👥 팀 정보

**팀명:** TriGuard AI  
**팀장:** 김기호 (한국공학대학교)  
**팀원:** 최희찬 (기획/전략), 김태희 (자료조사/데이터수집/정리, 충북대학교)
