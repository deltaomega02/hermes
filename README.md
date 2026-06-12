# HERMES

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white) ![SQLite](https://img.shields.io/badge/SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white) ![Google Cloud](https://img.shields.io/badge/Google_Cloud-4285F4?style=for-the-badge&logo=googlecloud&logoColor=white) ![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonwebservices&logoColor=white)

Bybit USDT 무기한 선물 멀티코인 자동매매 시스템.
4H 레짐 판독 + 1H 추세 풀백 진입 + 실시간 트레일링 스탑. 개인 프로젝트로 설계부터 백테스트, 클라우드 운영까지 단독 수행했다.

## 개요

4개 코인(BTC, ETH, SOL, XRP)을 동시 감시한다. 4H 캔들로 시장 레짐을 분류하고, 1H 캔들에서 EMA 풀백 시그널을 평가한다. 오더북 불균형과 펀딩레이트로 방향을 확인한 뒤 시장가로 진입한다. WebSocket으로 실시간 SL/TP를 관리하며, 트레일링 스탑으로 수익을 확정한다. 최대 3개 포지션 동시 보유.

설계상 핵심은 트레일링 스탑이다. 백테스트 6년 구간에서 TP 풀히트는 10회에 불과했고, 수익의 대부분은 트레일링 익절(3,537회)에서 나왔다. 실제 R:R 0.88, 승률 57%로 손익비가 아닌 승률로 누적하는 구조다.

## 아키텍처

```
               +-------------+
               |   main.py   |
               |  (멀티코인)  |
               +------+------+
                      |
         +------------+------------+
         |            |            |
  +------v------+ +---v--------+ +-v-----------+
  | regime_engine| | signal_engine| | risk_manager|
  | (4H, 코인별) | | (1H, 코인별) | | (사이징)    |
  | ADX/ATR/EMA | | EMA/RSI/BB  | | 드로다운    |
  +------+------+ +---+--------+ +-+-----------+
         |            |            |
         +-----+------+-----+-----+
               |             |
        +------v------+ +---v-----------+
        | orderbook   | | position_mgr  |
        | funding_rate| | (시장가 실행)  |
        +-------------+ +---+-----------+
                             |
              +--------------+--------------+
              |              |              |
       +------v------+ +----v----+ +------v------+
       | bybit_client | | bybit_ws| | db_manager  |
       | (REST API)   | | (실시간) | | (SQLite)    |
       +--------------+ +---------+ +-------------+
```

## 전략

### 추세 풀백 (TRENDING 레짐)

유일한 활성 전략. 레인지 반전 전략은 백테스트 -93%로 비활성 처리했다.

4H에서 ADX+EMA로 추세를 확인하고, 1H에서 EMA9 풀백을 기다린다:

1. 4H 레짐 TRENDING (ADX >= 30)
2. 1H 가격이 EMA9 근처 (풀백 거리 -0.1% ~ +1.5%)
3. EMA 배열 일치 (LONG: EMA9 > EMA21, SHORT: EMA9 < EMA21)
4. 오더북 불균형 방향 확인 (55% 이상)
5. 스코어 합산 >= 40점 (RSI + 볼륨 + 펀딩레이트)

시장가 진입. SL: 1.5x ATR. TP: 6.0x SL. 트레일링 스탑 1.2%/0.1%.
레버리지: ADX >= 30이면 최대 7x, 미만이면 3x.

### 거래 중단 (HIGH_VOL 레짐)

ATR 퍼센타일 85% 이상 또는 1H 캔들 3% 이상 변동 시 자동 전환. 1시간 쿨다운 후 복귀.

## 리스크 관리

| 항목 | 설정 |
|------|------|
| 거래당 리스크 | 잔액의 1.5% |
| 동시 포지션 | 최대 3개 |
| 일일 손실 한도 | 3% 초과 시 중단 |
| 드로다운 경고 | 5% (사이즈 50% 축소) |
| 드로다운 셧다운 | 20% |
| 펀딩비 회피 | 정산 3분 전 진입 차단 (00/08/16 UTC) |
| 트레일링 스탑 | 수익 1.2% 도달 시 활성화, 0.1% 거리 추적 |

연패 쿨다운은 백테스트에서 수익을 절반으로 깎는 것이 확인되어 제거했다. (추세 풀백 특성상 SL 직후가 오히려 좋은 재진입 자리)

## 데이터 소스

| 소스 | 방식 | 용도 |
|------|------|------|
| 4H OHLCV | REST `/v5/market/kline` | 레짐 분류 (ADX, ATR, EMA, MACD) |
| 1H OHLCV | REST `/v5/market/kline` | 진입 시그널 (EMA, RSI, BB, ATR) |
| 오더북 | REST `/v5/market/orderbook` | 매수/매도 불균형 확인 |
| 펀딩레이트 | REST `/v5/market/tickers` | 방향 편향 |
| Mark Price | WebSocket `tickers` | 실시간 SL/TP 감시 |
| 포지션 | WebSocket `position` | 서버 청산 감지 |

## 파라미터

주요 값은 `config/tunable_params.json`에서 관리한다.

| 파라미터 | 값 | 설명 |
|----------|-----|------|
| `ema_fast` / `ema_slow` | 3 / 15 | 1H EMA 페어 (14k 조합 sweep 검증) |
| `sl_atr_mult` | 1.5 | SL = ATR x 1.5 |
| `tp_rr_ratio` | 6.0 | TP = SL x 6.0 |
| `entry_score_threshold` | 40 | 최소 진입 스코어 |
| `adx_enter_trending` | 30 | 추세 진입 ADX 기준 |
| `trailing_activation_pct` | 1.2 | 트레일링 활성화 (%) |
| `trailing_distance_pct` | 0.1 | 트레일링 추적 거리 (%) |

50거래 축적 후 Optuna Walk-Forward 최적화를 주 1회 자동 실행한다. Train 200건 / Validation 100건을 분리하고, Validation에서 악화되면 폐기한다.

## 백테스트 검증

6년(2020-03 ~ 2026-04) 데이터, 슬리피지 0.05%와 서버 비용을 반영해 검증했다.

| 항목 | 결과 |
|------|------|
| 거래 수 | 6,208회 |
| 승률 | 57.2% |
| 실제 R:R | 0.88 |
| 최대 드로다운 | 52.6% |
| 트레일링 익절 / TP 풀히트 | 3,537회 / 10회 |

- 슬리피지 민감도: 0.05%에서 생존, 0.08%에서 파산 — 마진이 얇은 전략임을 인지하고 운영.
- 백테스트 단일 경로 수익은 상한치로 보고, Monte Carlo 시뮬레이션의 보수적 기대값을 운영 기준으로 삼았다.
- 실전 운영 9일과 백테스트를 비교한 결과 승률·손익 방향·최악 구간이 일치해, 엔진이 현실을 반영함을 확인했다.

상세한 백테스트 과정과 버전별 결과는 [hermes-backtesting](https://github.com/deltaomega02/hermes-backtesting) 저장소 참고.

## 실행

요구사항: Python 3.10+, Bybit 계정(서브계정 권장), 텔레그램 봇(선택)

```bash
# 환경변수 (.env)
BYBIT_API_KEY=
BYBIT_SECRET=
BYBIT_USE_TESTNET=false
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=

pip install -r requirements.txt
python main.py
```

GCP e2-small(서울 리전)에서 24시간 운영했다 (현재는 후속 세대로 이전). `nohup python3 -u main.py > ./logs/output.log 2>&1 &`

## 프로젝트 구조

```
hermes/
├── main.py                      # 멀티코인 메인 루프
├── config/                      # 설정, 튜닝 파라미터
├── core/
│   ├── regime_engine.py         # 4H 레짐 분류 (ADX/ATR/EMA/MACD)
│   ├── signal_engine.py         # 1H 추세 풀백 시그널
│   ├── risk_manager.py          # 포지션 사이징, 드로다운, 일일 한도
│   ├── position_manager.py      # 주문 실행, SL/TP 설정
│   └── websocket_watcher.py     # 실시간 가격 감시, 트레일링 스탑
├── exchange/                    # Bybit REST/WebSocket 클라이언트
├── database/                    # SQLite (거래 기록)
├── backtest/                    # 백테스트 엔진, Walk-Forward Optuna
└── utils/                       # 텔레그램 알림
```

## 버전 히스토리

| 버전 | 변경 |
|------|------|
| v3 | 트레일링 스탑 도입 |
| v4 | Shared-balance 백테스트 엔진, 쿨다운 제거 |
| v5 | 트레일링 재최적화 (1.2/0.1), 슬리피지 검증 |
| v6 | 4코인(+XRP), 3포지션, 레버리지 7x |
| v7 | 일일 거래 한도 제거 |
| v8 | EMA 5/18 → 3/15 (14k 조합 sweep 검증) |

## 면책

연구·학습 목적의 개인 프로젝트입니다. 암호화폐 선물은 고위험 상품이며, 과거 성과가 미래 수익을 보장하지 않습니다.
