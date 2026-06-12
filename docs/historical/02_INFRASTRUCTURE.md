# 02. 인프라 (GCP)

## 서버 정보

```
인스턴스 이름:  metis-bot-server
머신 유형:      e2-small (vCPU 2, 2GB Memory)
리전:           asia-northeast3 (Seoul, South Korea)
운영체제:       Ubuntu 22.04 LTS (Jammy)
고정 IP:        YOUR_SERVER_IP
디스크:         29GB (현재 50% 사용)
스왑:           4GB (256KB만 사용 중, 거의 미사용)
월 비용:        ~$23 (₩34,500)
```

## 현재 리소스 사용 (2026-04-26 측정)

```
RAM 사용:       309MB / 1.9GB (16%)
스왑:           4GB 대기 (0.0% 사용)
CPU load:       0.00 (완전 유휴)
디스크:         15GB / 29GB
```

### 프로세스별 메모리
| 프로세스 | 메모리 | CPU |
|---|---|---|
| Google Ops Agent | 135MB | 0.2% |
| **HERMES (main.py)** | **86MB** | 0.4% |
| fluent-bit | 22MB | 0.2% |
| 기타 시스템 | ~50MB | 미미 |

## UPBIT_BOT 추가 시 예상

```
UPBIT_BOT 추정:        +100MB (HERMES 유사 구조)
                       +0.5% CPU
                       +1GB 디스크 (DB + 로그)
─────────────────────
추가 후 총:
  RAM:   409MB / 1.9GB (22% 사용, 여유 1.4GB)   CPU:   ~1% 평상시   디스크: 16GB / 29GB ```

**판정**: e2-small에 UPBIT_BOT 추가해도 **여유 압도적**. 추가 봇 2~3개도 가능.

## 권장 디렉토리 구조 (GCP 서버)

```
~/                              (홈, scp 업로드 임시 위치)
├── hermes-trading/             (HERMES, 기존)
│   ├── main.py
│   ├── config/
│   ├── core/
│   ├── exchange/
│   ├── utils/
│   ├── database/
│   ├── logs/
│   └── hermes/                 (Python venv)
│
└── upbit-bot/                  (UPBIT_BOT, 신규)
    ├── main.py
    ├── config/
    ├── core/
    ├── exchange/               (업비트 API)
    ├── utils/
    ├── database/
    ├── logs/
    └── upbit/                  (별도 Python venv)
```

**중요**: 별도 Python venv 권장 (의존성 충돌 방지).

## 배포 워크플로우 (HERMES와 동일)

```bash
# 1단계: scp로 GCP 홈에 업로드
cd /Users/sue/Projects/UPBIT_BOT/code
scp main.py                      [GCP서버]:~/
scp config/settings.py           [GCP서버]:~/
# ... 다른 파일들

# 2단계: GCP에서 mv
mv ./main.py            ./upbit-bot/main.py
mv ./settings.py        ./upbit-bot/config/settings.py
# ...

# 3단계: 재시작
pkill -f "python3 -u main.py"  # ← 주의: HERMES도 같은 패턴이라 신중
# 더 안전: 별도 systemd service 또는 다른 진입점 이름
```

**중요 주의**: `pkill -f "python3 -u main.py"` 명령은 HERMES와 UPBIT_BOT 둘 다 죽일 수 있음.

### 안전한 재시작 방법
```bash
# UPBIT_BOT만 종료
pkill -f "python3 -u /home/botuser/upbit-bot/main.py"

# 또는 PID 직접 관리
ps aux | grep upbit-bot
kill <PID>
```

또는 **다른 진입점 이름**:
- HERMES: `main.py`
- UPBIT_BOT: `upbit_main.py` 같이 다른 이름

## 모니터링 명령어

리소스 체크 (HERMES 메모리에 있던 명령):
```bash
echo "=== 메모리 + 스왑 ===" && free -h && \
echo "" && echo "=== 스왑 상세 ===" && swapon --show && \
echo "" && echo "=== 디스크 ===" && df -h / && \
echo "" && echo "=== 부하 평균 ===" && uptime && \
echo "" && echo "=== 메모리 상위 10 프로세스 ===" && \
ps aux --sort=-%mem | head -11 && \
echo "" && echo "=== CPU 상위 10 프로세스 ===" && \
ps aux --sort=-%cpu | head -11
```

## HERMES와 충돌 회피

1. **포트 분리**: 두 봇이 같은 포트 사용 X (WebSocket 자동 할당이라 보통 OK)
2. **로그 분리**: 각자 `logs/` 디렉토리
3. **DB 분리**: HERMES `hermes.db`, UPBIT_BOT `upbit.db`
4. **API 키 분리**: 각자 `.env`
5. **텔레그램 봇 분리**: 채널/봇 다르게 (혼동 방지)

## 준비 항목

- [ ] 업비트 API 키 발급 및 권한 확인
- [ ] e2-small 리소스 다시 측정 (변동 확인)
- [ ] Python venv 별도 설치
- [ ] 텔레그램 새 봇 생성 + 토큰
- [ ] DB SQLite 또는 PostgreSQL 결정
- [ ] systemd 서비스 파일 작성 (자동 재시작)
- [ ] 백테스트 데이터 (업비트 OHLCV) 확보
