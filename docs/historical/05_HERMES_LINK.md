# 05. HERMES 연계

## HERMES와의 관계

**원칙**: 완전 별도 프로젝트. 자금/코드/DB/텔레그램 분리. 단 같은 서버 + 같은 운영자 + 같은 수동 감독.

## 코드 재활용 가능성

### HERMES 모듈별 재활용도

| HERMES 모듈 | UPBIT_BOT 재활용 | 비고 |
|---|---|---|
| `core/regime_engine.py` | **그대로** | 4H ADX/EMA/MACD 레짐 판정 |
| `core/technical_analysis.py` | **그대로** | EMA/RSI/BB/ATR/ADX 계산 |
| `core/signal_engine.py` | **수정 필요** | LONG only, SHORT 로직 제거 |
| `core/risk_manager.py` | **수정 필요** | 레버리지 X, 마진 X |
| `core/position_manager.py` | **재작성** | 업비트 API 다름 |
| `core/websocket_watcher.py` | **재작성** | 업비트 WebSocket 다름 |
| `exchange/bybit_client.py` | **재작성 (upbit_client.py)** | 업비트 API |
| `database/db_manager.py` | **그대로** | DB 스키마 동일 |
| `utils/telegram_bot.py` | **그대로** (봇 토큰만 변경) | 메시지 포맷 재활용 |
| `config/parameters.py` | **단순화** | 트레일링/SL ATR 등 일부만 |
| `config/settings.py` | **수정** | 거래소/심볼 변경 |

### 옵션별 재활용 비율
- **옵션 A (DCA)**: HERMES 코드 ~30% 재활용 (인프라/DB/알림만)
- **옵션 B (Long-Only HERMES)**: ~70% 재활용 (전략 그대로)
- **옵션 C (Grid)**: ~20% 재활용 (Grid는 HERMES에 없음)

## 업비트 API (vs Bybit)

### Bybit API (HERMES 사용 중)
- REST: `api.bybit.com/v5/`
- WebSocket: `stream.bybit.com/v5/`
- 인증: HMAC SHA256
- v5 통합 API

### 업비트 API
- REST: `api.upbit.com/v1/`
- WebSocket: `api.upbit.com/websocket/v1`
- 인증: JWT
- Python 라이브러리: `pyupbit` (간단)

### 주요 엔드포인트 매핑
| 기능 | Bybit | 업비트 |
|---|---|---|
| 잔고 조회 | `/v5/account/wallet-balance` | `/v1/accounts` |
| 시세 | `/v5/market/tickers` | `/v1/ticker` |
| OHLCV | `/v5/market/kline` | `/v1/candles/{minutes}` |
| 매수 | `/v5/order/create` | `/v1/orders` |
| 매도 | `/v5/order/create` | `/v1/orders` |
| 주문 조회 | `/v5/order/realtime` | `/v1/order` |

### 업비트 권장 라이브러리
```python
# pip install pyupbit
import pyupbit

upbit = pyupbit.Upbit(access_key, secret_key)
balance = upbit.get_balance("KRW")  # 원화 잔고
upbit.buy_market_order("KRW-BTC", 50000)  # ₩50,000 BTC 시장가
```

매우 단순. HERMES Bybit API 추상화보다 짧음.

## 데이터 공유 (운영자 감독 일원화)

### 일일 통합 리포트 (제안)
```
매일 09:00 KST 텔레그램 통합 메시지:
  운영자 자동매매 일일 요약
  ────────────────
  HERMES (Bybit):
    잔고 $XXX / DD X% / 일PnL $XXX
  UPBIT_BOT:
    잔고 ₩XXX / DD X% / 일PnL ₩XXX
  ────────────────
  통합 PnL: ₩XXX
```

이걸 위해 별도 통합 봇 또는 Claude 분석 스크립트 가능.

### CSV 통합 기록
```
HERMES_실전기록/실전거래기록.csv (기존)
UPBIT_BOT_실전기록/실전거래기록.csv (신규)
```

운영자가 두 시스템 결과를 같은 운영자 Claude 세션에서 분석 가능.

## 운영 충돌 방지

### 1. 프로세스 분리
HERMES와 UPBIT_BOT 둘 다 `python3 main.py`로 실행하면 `pkill -f` 시 둘 다 죽음.

**해결책 A**: 다른 진입점 이름
- HERMES: `main.py`
- UPBIT_BOT: `upbit_main.py` 또는 `start.py`

**해결책 B**: systemd 서비스로 분리
```bash
# /etc/systemd/system/hermes.service
# /etc/systemd/system/upbit-bot.service
```

### 2. API 한도 분리
- HERMES Bybit API: 영향 없음
- UPBIT_BOT 업비트 API: 영향 없음
- 둘이 다른 거래소라 충돌 X

### 3. 텔레그램 봇 분리
**필수**: HERMES 봇과 별도 봇 (혼동 방지)
- HERMES 봇: 기존 사용 중
- UPBIT_BOT 봇: 신규 생성 필요

### 4. 디스크 공간
GCP e2-small 30GB 중 50% (15GB) 사용 중. UPBIT_BOT 추가 후 +1~2GB 예상. 여유 충분.

## HERMES 운영 경험에서 배운 것 (UPBIT_BOT 적용)

### 적용 사항
1. **SL/TP 서버 등록** (재시작 무관 안전망)
2. **DD 단계화 사이징**
3. **Bid/Ask Limit** (수수료 절감) — 업비트는 maker 우대 X라 적용도 미미
4. **점수 임계 60+ 강신호만** (V13.1에서 검증)
5. **30분 누적 요약 알림**
6. **OrderLinkID 중복 방지** (업비트 identifier 활용)

### 적용 안 함
1. **트레일링 스탑** (현물 + LONG only는 단순 SL/TP가 자연)
2. **펀딩 회피** (업비트 펀딩 없음)
3. **연속 패배 차단** (HERMES에서 제거됨)
4. **레짐 멀티 분기** (V14에서 검증 후 롤백)

## 구현 단계 (옵션 A 가정)

### 코드 개발
- 업비트 API 키 발급
- pyupbit 라이브러리 학습
- DCA 코드 작성 (간단)
- 텔레그램 봇 새로 생성
- GCP 디렉토리 구조 준비

### 가동
- 시드 ₩100만 입금
- 페이퍼 모드 검증
- 실전 가동
- 첫 매수 결과 확인

### 운영
- 일일 감독
- 매수 결과 추적
- HERMES와 통합 리포트
