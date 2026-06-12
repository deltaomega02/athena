# ATHENA

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white) ![Google Gemini](https://img.shields.io/badge/Google_Gemini-8E75B2?style=for-the-badge&logo=googlegemini&logoColor=white) ![Google Cloud](https://img.shields.io/badge/Google_Cloud-4285F4?style=for-the-badge&logo=googlecloud&logoColor=white) ![SQLite](https://img.shields.io/badge/SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white)

AI 포트폴리오 매니저 방식의 암호화폐 현물 자동운용 시스템. (운영 중지 — 현행 운영은 [metis](https://github.com/deltaomega02/metis))

앞선 6세대 시스템(VALKYR → ARGOS → OMNI → METIS → HERMES → KAIROS)에서 검증된 패턴만 모아 재설계했다. 핵심 차이는 AI의 역할이다 — 매매 시그널 생성기가 아니라, **비중을 결정하는 포트폴리오 매니저**로 제한하고 실행과 안전장치는 전부 코드가 담당한다.

## 설계 원칙

| 역할 | 담당 |
|---|---|
| 시장 분석·비중 결정 | AI (Multi-Agent: Analyst / Risk Manager / Portfolio Manager) |
| 주문 실행 | 코드 (Bybit Spot API) |
| 안전장치 | 코드 (리밸런싱 threshold 15%, 드로다운 -15% 셧다운) |

이전 세대에서 가져온 것:

- 다단계 목표가 컨셉 (VALKYR)
- AI Chain-of-Thought 추론 (ARGOS)
- 회고 기반 학습 루프 (OMNI)
- 단계별 분석 파이프라인 (METIS)
- Bybit API·WebSocket·DB 인프라 (HERMES)
- 단순화·버전 동결 운영 원칙 (KAIROS)

## 동작 방식

4시간 주기로 동작한다.

1. **데이터 수집** — 무료 공개 API 8종에서 통합 컨텍스트 생성
2. **Multi-Agent 분석** — Analyst가 시장 분석 → Risk Manager가 리스크 평가 → Portfolio Manager가 목표 비중 결정
3. **리밸런싱** — 목표 비중과 현재 비중의 괴리가 threshold(15%)를 넘을 때만 주문 실행
4. **기록** — 모든 결정과 근거를 DB에 저장, 회고에 활용

### 데이터 입력 (전부 무료 API)

| 카테고리 | 출처 |
|---|---|
| 가격/펀딩/OI | Bybit, Coinalyze |
| 센티먼트 | Alternative.me Fear & Greed |
| 거시 지표 (DXY/SPX) | yfinance |
| ETF 자금 흐름 | Farside Investors |
| 뉴스/소셜 | cryptocurrency.cv, Reddit JSON |
| 김치 프리미엄 | kimpga |
| 온체인 | DefiLlama |

## 운영 원칙

1. 파라미터 동결 — 50거래까지 변경 금지
2. 단순화 우선 — 지표·필터 추가 충동 차단
3. 검증 단계 분리 — 페이퍼 모드 → 결정 일관성 평가 → 소액 실전 → 검증 후 확대
4. AI는 매니저, 코드는 실행자 — 역할 경계를 코드로 강제
5. 결과를 그대로 기록 — 벤치마크(단순 보유)보다 못하면 그대로 인정

## 실행

```bash
# 환경변수 (.env)
BYBIT_API_KEY=
BYBIT_SECRET=
GEMINI_API_KEY=
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=

pip install -r requirements.txt
python main.py
```

GCP e2-small에서 systemd 서비스(`athena.service`)로 운영하며 텔레그램으로 결정 내역을 보고받는 구조로 설계했다.

## 구조

```
athena/
├── main.py              # 메인 루프 (4H 주기)
├── ai/                  # Multi-Agent 프롬프트, Gemini 클라이언트
├── core/                # 데이터 수집, 리밸런싱, 안전장치
├── data_sources/        # 무료 API 어댑터 8종
├── exchange/            # Bybit Spot 클라이언트
├── database/            # 결정·거래 기록 (SQLite)
├── utils/               # 텔레그램 알림
├── dashboard.py         # 운영 대시보드
└── athena.service       # systemd 유닛
```

## 면책

연구·학습 목적의 개인 프로젝트입니다. 금융 자문이 아니며, 과거 성과가 미래 수익을 보장하지 않습니다.
