# NanoClaw 멀티에이전트 구성 가이드

## 개요

하나의 Mac에서 여러 NanoClaw 인스턴스를 실행하여 **각각 독립된 봇 아이덴티티**를 가진 멀티에이전트 시스템을 구성한다.

```
┌─────────────────────────────────────────────────────┐
│                    macOS 호스트                       │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ NanoClaw A   │  │ NanoClaw B   │  │ NanoClaw C │ │
│  │ "Nano"       │  │ "Jarvis"     │  │ "Trader"   │ │
│  │ #xclaw       │  │ #jarvis-dev  │  │ #trading-* │ │
│  │ 범용 에이전트 │  │ 개발 에이전트 │  │ 트레이딩   │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                │         │
│         └────────┬────────┴────────┬───────┘         │
│                  │                 │                  │
│         ┌───────────────┐  ┌─────────────┐           │
│         │ OneCLI 프록시  │  │ Docker/     │           │
│         │ (공유, 1개)    │  │ Colima(공유)│           │
│         └───────────────┘  └─────────────┘           │
└─────────────────────────────────────────────────────┘
```

## 공유 vs 개별 컴포넌트

| 컴포넌트 | 공유/개별 | 이유 |
|----------|----------|------|
| **Docker/Colima** | 공유 | 컨테이너 런타임은 하나면 충분 |
| **OneCLI 게이트웨이** | 공유 | 동일 API 크레덴셜 사용 시 1개로 충분 |
| **Node.js** | 공유 | 동일 바이너리 |
| **nanoclaw-agent 이미지** | 공유 | 동일 컨테이너 이미지 |
| **프로젝트 디렉토리** | **개별** | 각 인스턴스 독립 설정 |
| **.env** | **개별** | 다른 Slack 봇 토큰 |
| **Slack 앱** | **개별** | 다른 봇 이름/아바타 |
| **launchd plist** | **개별** | 다른 서비스 이름 |
| **store/messages.db** | **개별** | 독립 대화 이력 |
| **groups/** | **개별** | 독립 메모리/성격 |

## 디렉토리 구조

```
~/Project/xclaw/
├── nanoclaw/              # 인스턴스 A: Nano (원본 fork)
│   ├── .env               # SLACK_BOT_TOKEN=xoxb-nano-...
│   ├── groups/
│   │   └── slack_xclaw/
│   └── store/messages.db
│
├── nanoclaw-jarvis/       # 인스턴스 B: Jarvis
│   ├── .env               # SLACK_BOT_TOKEN=xoxb-jarvis-...
│   ├── groups/
│   │   └── slack_jarvis-dev/
│   └── store/messages.db
│
└── nanoclaw-trader/       # 인스턴스 C: Trader
    ├── .env               # SLACK_BOT_TOKEN=xoxb-trader-...
    ├── groups/
    │   └── slack_trading/
    └── store/messages.db
```

## 구성 단계

### Step 1: 추가 인스턴스 복제

```bash
# 원본 fork에서 복제 (git clone이 아닌 cp — 각 인스턴스가 독립)
cp -r ~/Project/xclaw/nanoclaw ~/Project/xclaw/nanoclaw-jarvis
cp -r ~/Project/xclaw/nanoclaw ~/Project/xclaw/nanoclaw-trader

# 각 인스턴스의 DB/세션/로그 초기화
for dir in nanoclaw-jarvis nanoclaw-trader; do
  cd ~/Project/xclaw/$dir
  rm -rf store/ data/sessions/ logs/
  mkdir -p store data/sessions logs data/env data/certs
  # OneCLI CA cert 공유 (심볼릭 링크)
  ln -sf ~/Project/xclaw/nanoclaw/data/certs/onecli-ca.pem data/certs/onecli-ca.pem
done
```

### Step 2: 각 인스턴스별 Slack 앱 생성

[api.slack.com/apps](https://api.slack.com/apps) 에서 **각 에이전트별 Slack 앱**을 생성:

| 인스턴스 | Slack 앱 이름 | 용도 |
|---------|-------------|------|
| A | Nano | 범용 |
| B | Jarvis | 개발 전문 |
| C | Trader | 트레이딩 전문 |

각 앱마다:
1. Socket Mode 활성화 → App Token (`xapp-...`)
2. Event Subscriptions → `message.channels`, `message.groups`, `message.im`
3. OAuth Scopes → `chat:write`, `channels:history`, `groups:history`, `im:history`, `channels:read`, `groups:read`, `users:read`
4. Install to Workspace → Bot Token (`xoxb-...`)

### Step 3: 각 인스턴스 .env 설정

```bash
# nanoclaw-jarvis/.env
cat > ~/Project/xclaw/nanoclaw-jarvis/.env << 'EOF'
SLACK_BOT_TOKEN=xoxb-jarvis-봇-토큰
SLACK_APP_TOKEN=xapp-jarvis-앱-토큰
ASSISTANT_NAME=Jarvis
TZ=Asia/Seoul
ONECLI_URL=http://127.0.0.1:10254
EOF

# nanoclaw-trader/.env
cat > ~/Project/xclaw/nanoclaw-trader/.env << 'EOF'
SLACK_BOT_TOKEN=xoxb-trader-봇-토큰
SLACK_APP_TOKEN=xapp-trader-앱-토큰
ASSISTANT_NAME=Trader
TZ=Asia/Seoul
ONECLI_URL=http://127.0.0.1:10254
EOF
```

환경 동기화:
```bash
for dir in nanoclaw-jarvis nanoclaw-trader; do
  mkdir -p ~/Project/xclaw/$dir/data/env
  cp ~/Project/xclaw/$dir/.env ~/Project/xclaw/$dir/data/env/env
done
```

### Step 4: 그룹 등록

각 인스턴스에서 Claude Code 실행 → 그룹 등록:

```bash
# Jarvis 인스턴스
cd ~/Project/xclaw/nanoclaw-jarvis
PATH="/opt/homebrew/opt/node@22/bin:$PATH" npx tsx setup/index.ts --step environment
PATH="/opt/homebrew/opt/node@22/bin:$PATH" npx tsx setup/index.ts --step register \
  -- --jid "slack:C채널ID대문자" --name "jarvis-dev" \
  --folder "slack_jarvis-dev" --trigger "@Jarvis" \
  --channel slack --assistant-name "Jarvis" \
  --is-main --no-trigger-required
```

**주의: 채널 ID는 반드시 대문자** (`C0AL2FVP4JJ` 등)

### Step 5: 각 인스턴스 CLAUDE.md 커스터마이징

각 에이전트의 성격/역할을 정의:

```bash
# Jarvis — 개발 전문
cat > ~/Project/xclaw/nanoclaw-jarvis/groups/slack_jarvis-dev/CLAUDE.md << 'EOF'
# Jarvis

시니어 소프트웨어 엔지니어 에이전트. 코드 리뷰, 아키텍처 설계, 디버깅 전문.
항상 한국어로 응답. "yoonhwan 오빠" 호칭 사용.

## 전문 분야
- Python, TypeScript, Rust
- 시스템 아키텍처
- 코드 리뷰
- 디버깅
EOF

# Trader — 트레이딩 전문
cat > ~/Project/xclaw/nanoclaw-trader/groups/slack_trading/CLAUDE.md << 'EOF'
# Trader

트레이딩 분석 에이전트. 시장 데이터 분석, 전략 백테스트, 포지션 관리.
항상 한국어로 응답.

## 전문 분야
- 기술적 분석 (TA)
- 리스크 관리
- 매매 일지 작성
- 전략 검증
EOF
```

### Step 6: launchd 서비스 등록

각 인스턴스별 별도 plist 생성:

```bash
# Jarvis plist
cat > ~/Library/LaunchAgents/com.nanoclaw-jarvis.plist << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.nanoclaw-jarvis</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/opt/node@22/bin/node</string>
        <string>$HOME/Project/xclaw/nanoclaw-jarvis/dist/index.js</string>
    </array>
    <key>WorkingDirectory</key>
    <string>$HOME/Project/xclaw/nanoclaw-jarvis</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/opt/homebrew/opt/node@22/bin:/usr/local/bin:/usr/bin:/bin:$HOME/.local/bin</string>
        <key>HOME</key>
        <string>$HOME</string>
        <key>DOCKER_HOST</key>
        <string>unix://$HOME/.colima/default/docker.sock</string>
    </dict>
    <key>StandardOutPath</key>
    <string>$HOME/Project/xclaw/nanoclaw-jarvis/logs/nanoclaw.log</string>
    <key>StandardErrorPath</key>
    <string>$HOME/Project/xclaw/nanoclaw-jarvis/logs/nanoclaw.error.log</string>
</dict>
</plist>
EOF
```

**$HOME을 실제 경로로 치환 필요** (`/Users/yoonhwan`)

서비스 시작:
```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw-jarvis.plist
launchctl load ~/Library/LaunchAgents/com.nanoclaw-trader.plist
```

### Step 7: Slack에서 봇 추가 + 테스트

각 채널에 해당 봇만 추가:
- #xclaw → Nano 봇 추가
- #jarvis-dev → Jarvis 봇 추가
- #trading-dev → Trader 봇 추가

## 관리 명령어

### 전체 상태 확인
```bash
launchctl list | grep nanoclaw
# com.nanoclaw          (PID) 0   ← Nano
# com.nanoclaw-jarvis   (PID) 0   ← Jarvis
# com.nanoclaw-trader   (PID) 0   ← Trader
```

### 개별 재시작
```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw-jarvis
```

### 전체 재시작
```bash
for svc in com.nanoclaw com.nanoclaw-jarvis com.nanoclaw-trader; do
  launchctl kickstart -k gui/$(id -u)/$svc
done
```

### 로그 확인
```bash
# 모든 인스턴스 로그 동시 확인
tail -f ~/Project/xclaw/nanoclaw/logs/nanoclaw.log \
       ~/Project/xclaw/nanoclaw-jarvis/logs/nanoclaw.log \
       ~/Project/xclaw/nanoclaw-trader/logs/nanoclaw.log
```

## 리소스 사용량

| 항목 | 인스턴스당 | 3개 인스턴스 |
|------|-----------|-------------|
| Node.js 프로세스 | ~50MB RAM | ~150MB RAM |
| 컨테이너 (활성 시) | ~200MB RAM | 최대 ~600MB (동시 활성) |
| 디스크 | ~100MB | ~300MB |
| Docker 이미지 | 공유 (1개) | ~500MB (1회) |

**총 예상**: 유휴 시 ~200MB, 3개 동시 응답 시 ~800MB

## 업데이트 전략

원본 fork만 업데이트 → 다른 인스턴스에 동기화:

```bash
# 1. 원본 업데이트
cd ~/Project/xclaw/nanoclaw
git fetch upstream && git merge upstream/main

# 2. 빌드
PATH="/opt/homebrew/opt/node@22/bin:$PATH" npm install && npm run build

# 3. 다른 인스턴스에 dist/ 동기화 (설정 파일 제외)
for dir in nanoclaw-jarvis nanoclaw-trader; do
  rsync -av --exclude='.env' --exclude='store/' --exclude='groups/' \
    --exclude='data/' --exclude='logs/' --exclude='node_modules/' \
    ~/Project/xclaw/nanoclaw/ ~/Project/xclaw/$dir/
  cd ~/Project/xclaw/$dir
  PATH="/opt/homebrew/opt/node@22/bin:$PATH" npm install && npm run build
done

# 4. 컨테이너 이미지 재빌드 (공유)
cd ~/Project/xclaw/nanoclaw
DOCKER_HOST=unix://$HOME/.colima/default/docker.sock ./container/build.sh

# 5. 전체 재시작
for svc in com.nanoclaw com.nanoclaw-jarvis com.nanoclaw-trader; do
  launchctl kickstart -k gui/$(id -u)/$svc
done
```

## xClaw과의 관계

이 멀티에이전트 구성은 **수동 오케스트레이션**이다.
xClaw(L1 OpenClaw)의 목표는 이걸 **자동화**하는 것:

```
현재 (수동):
  사람이 각 인스턴스 관리, 채널 할당, 업데이트 동기화

xClaw 목표 (자동):
  L1 오케스트레이터 → NanoClaw 인스턴스 자동 스폰/관리
                    → 태스크 자동 라우팅
                    → 크루 간 통신 (Redis Streams)
                    → 헬스체크 + 자동 복구
```

## 주의사항

1. **컨테이너 이름 충돌**: 각 인스턴스가 `nanoclaw-slack-{group}-{ts}` 패턴으로 컨테이너 생성. 그룹명이 다르면 충돌 없음
2. **OneCLI 포트**: 모든 인스턴스가 같은 OneCLI(10254/10255) 공유. 에이전트별 rate limit 필요 시 OneCLI 대시보드에서 설정
3. **동시 컨테이너**: 기본 `MAX_CONCURRENT_CONTAINERS=5`가 인스턴스별 적용. 3개 인스턴스 × 5 = 최대 15개 동시 컨테이너 가능
4. **같은 채널에 여러 봇**: 가능하지만 권장하지 않음. 메시지 중복 처리됨
5. **Slack 봇 이름/아바타**: api.slack.com/apps → 각 앱의 Display Information에서 설정
