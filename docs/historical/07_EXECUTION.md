# 07. 가동 절차

## 준비 항목

### 인증/계정
- [ ] 업비트 회원가입 + KYC 완료
- [ ] 업비트 API 키 발급 (출금 권한 X)
- [ ] GCP IP 화이트리스트 등록 (`YOUR_SERVER_IP`)
- [ ] 텔레그램 새 봇 생성 (`@BotFather`)

### 코드 개발
- [ ] 전략 결정 (`04_STRATEGIES.md`)
- [ ] pyupbit 라이브러리 설치
- [ ] DCA 코드 작성 (옵션 A 가정)
  ```
  upbit-bot/
  ├── main.py              (메인 루프)
  ├── config/
  │   ├── settings.py      (시드, 매수 주기, 코인 비율)
  │   └── secrets.py       (.env 로드)
  ├── core/
  │   ├── scheduler.py     (DCA 스케줄)
  │   ├── trader.py        (매수 실행)
  │   └── monitor.py       (잔고 추적)
  ├── exchange/
  │   └── upbit_client.py  (pyupbit 래퍼)
  ├── utils/
  │   └── telegram_bot.py  (HERMES 코드 재활용)
  └── database/
      └── db_manager.py    (SQLite, HERMES 코드 재활용)
  ```
- [ ] 로컬 페이퍼 테스트 (실제 매수 X, 시뮬만)
- [ ] GCP 디렉토리 구조 준비

## 가동 절차

### 1단계: 입금
1. **업비트 ₩100만 입금** 확인
2. 잔고 조회 API 테스트
3. 텔레그램 알림 확인

### 2단계: 테스트
4. **소액 테스트 매수**: ₩5,000 BTC 시장가 매수
5. 체결 확인
6. 즉시 매도 (테스트 종료)

### 3단계: 실전
7. **실전 모드 전환** (.env에서 `PAPER_MODE=False`)
8. 첫 정식 매수: ₩50,000 (DCA 1회분)
9. 체결가/수수료 검증
10. 텔레그램 알림 도착 확인

### 4단계: 모니터링
11. 1일 동안 이상 없으면 정상 운영 선언
12. HERMES 영향 점검

## 평가 지표

| 지표 | 목표 | 우려 트리거 |
|---|---|---|
| 시드 보존 | ≥ -5% | -10% 초과 시 중단 |
| 매수 횟수 | DCA 정상 작동 | API 오류 1회 미만 |
| 텔레그램 알림 | 100% 도착 | 미도착 1회 시 점검 |
| HERMES 영향 | 0 (충돌 X) | 충돌 시 격리 |

### 결정 옵션
- **양호 → 시드 추가 입금 검토** (옵션 B 추가 또는 시드 ↑)
- **보통 → 운영 유지 + 누적 평가**
- **나쁨 → 전략 재검토 또는 일시 중단**

## GCP 배포 워크플로우

### scp 업로드
```bash
cd /Users/sue/Projects/UPBIT_BOT/code

scp main.py                          [GCP서버]:~/
scp config/settings.py               [GCP서버]:~/
scp core/*.py                        [GCP서버]:~/
scp exchange/upbit_client.py         [GCP서버]:~/
# ... 다른 파일들
```

### GCP에서 mv
```bash
mkdir -p ~/upbit-bot/config ~/upbit-bot/core ~/upbit-bot/exchange ~/upbit-bot/utils ~/upbit-bot/database ~/upbit-bot/logs

mv ./main.py                ./upbit-bot/main.py
mv ./settings.py            ./upbit-bot/config/settings.py
# ... 등
```

### Python 환경
```bash
cd ~/upbit-bot
python3 -m venv upbit
source upbit/bin/activate
pip install pyupbit pandas requests python-dotenv
```

### 가동
```bash
# 옵션 1: nohup
nohup python3 -u main.py > ./logs/output.log 2>&1 &

# 옵션 2: systemd 서비스 (권장)
sudo systemctl start upbit-bot
sudo systemctl enable upbit-bot
```

### systemd 서비스 파일
```ini
# /etc/systemd/system/upbit-bot.service
[Unit]
Description=UPBIT_BOT Auto Trading
After=network.target

[Service]
Type=simple
User=botuser
WorkingDirectory=/home/botuser/upbit-bot
Environment="PATH=/home/botuser/upbit-bot/upbit/bin"
ExecStart=/home/botuser/upbit-bot/upbit/bin/python3 -u main.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## 운영 모니터링

### 매일 점검
- [ ] 텔레그램 알림 도착 (매수 / 일일 요약)
- [ ] 업비트 잔고 vs 코드 기록 일치
- [ ] HERMES 영향 없는지 (메모리/CPU 모니터링)

### 매주 점검
- [ ] DCA 매수 정상 (월요일 09:00 자동 실행)
- [ ] 평균 매수가 vs 시장가 비교
- [ ] 수수료 누적
- [ ] CSV 거래 기록

### 알림 트리거
- 매수 실패 → 즉시 텔레그램
- API 오류 3회 연속 → 일시 중단 + 알림
- 잔고 -5% 이상 → 알림
- HERMES 충돌 감지 → 알림

## 운영자 결정 필요

1. **전략 옵션**: A / B / C / D 중?
2. **매수 주기**: 매주 / 매월 / 매일?
3. **매수 코인**: BTC / ETH / SOL / XRP / 기타?
4. **매수 비율**: 균등 / BTC 위주 / 기타?
5. **종료 조건**: 있나? (예: 50% 수익 시 일부 매도)

이 결정 받으면 코드 작성 가능.
