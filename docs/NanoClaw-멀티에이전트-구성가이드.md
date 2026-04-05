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

## NanoClaw 운영 스펙 (실측 기반)

### 참조 환경

| 항목 | 스펙 |
|------|------|
| **CPU** | Apple M3 Max (14코어) |
| **호스트 RAM** | 36 GB |
| **디스크** | 536 GB 여유 |
| **Colima VM** | CPU 2 / RAM 4 GB / 디스크 100 GB |
| **Docker 엔진** | Colima (macOS Virtualization.Framework) |

### 실측 리소스 사용량

| 컴포넌트 | 위치 | RAM 사용량 | 비고 |
|----------|------|-----------|------|
| **NanoClaw 호스트 프로세스** | 호스트 macOS | **64 MB** | Node.js, Slack 리스너, 큐 관리 |
| **OneCLI 게이트웨이** | Colima 내 | **110 MB** | 크레덴셜 프록시 |
| **OneCLI PostgreSQL** | Colima 내 | **51 MB** | 크레덴셜 저장소 |
| **에이전트 컨테이너 (활성)** | Colima 내 | **~250 MB** | Claude Agent SDK 실행 중 |
| **에이전트 이미지** | Colima 내 | — | **2.39 GB** (디스크, 공유) |
| **인스턴스 프로젝트 디렉토리** | 호스트 macOS | — | ~100 MB (디스크) |

### ⚠️ 핵심 병목: Colima VM RAM

에이전트 컨테이너는 **Colima VM 안에서** 실행된다.
호스트에 36GB가 있어도, Colima에 할당된 RAM만 사용 가능.

```
호스트 36 GB
  └── Colima VM: 4 GB (현재 설정)
        ├── OneCLI + PostgreSQL: ~160 MB (고정)
        ├── 기타 컨테이너 (vectoragent, langfuse 등): ~590 MB
        ├── Docker 오버헤드: ~200 MB
        └── 에이전트 컨테이너 가용: ~3 GB
```

### 최대 동시 에이전트 수 계산

#### 현재 설정 (Colima 4GB)

| 시나리오 | 계산 | 최대 |
|---------|------|------|
| 기타 컨테이너 없음 | (4GB - 360MB 오버헤드) / 250MB | **~14개** 동시 |
| 현재 환경 (기타 750MB) | (4GB - 360MB - 750MB) / 250MB | **~11개** 동시 |
| 안전 마진 80% | 11 × 0.8 | **~8개** 동시 (권장) |

#### Colima 확장 시

```bash
# Colima RAM 확장 (1회만 실행)
colima stop
colima start --cpu 4 --memory 8    # 8GB 할당
# 또는
colima start --cpu 6 --memory 16   # 16GB 할당
```

| Colima RAM | 기타 컨테이너 포함 | 최대 동시 에이전트 | 권장 |
|-----------|-------------------|-------------------|------|
| **4 GB** (현재) | ~750 MB | ~11개 | **8개** |
| **8 GB** | ~750 MB | ~27개 | **20개** |
| **16 GB** | ~750 MB | ~59개 | **45개** |
| **24 GB** | ~750 MB | ~91개 | **70개** |

### NanoClaw 인스턴스 수 vs 동시 에이전트

NanoClaw 인스턴스(호스트 프로세스)와 동시 에이전트 컨테이너는 다른 개념:

```
NanoClaw 인스턴스 = 호스트에서 실행 (64MB × N)
동시 에이전트 = Colima 안에서 실행 (250MB × N)
```

| NanoClaw 인스턴스 | 호스트 RAM | 인스턴스당 MAX_CONCURRENT | 최악 동시 컨테이너 | 필요 Colima RAM |
|------------------|-----------|--------------------------|-------------------|----------------|
| **3개** | 192 MB | 5 | 15개 | **~5 GB** |
| **5개** | 320 MB | 5 | 25개 | **~8 GB** |
| **10개** | 640 MB | 3 | 30개 | **~9 GB** |
| **20개** | 1.3 GB | 2 | 40개 | **~12 GB** |

> 💡 **실제로는** 모든 인스턴스가 동시에 max 컨테이너를 쓰진 않는다.
> 평상시 동시 활성 컨테이너는 전체의 10~20% 수준.

### 권장 구성

| 사용 패턴 | 인스턴스 수 | Colima 설정 | MAX_CONCURRENT |
|-----------|-----------|-------------|----------------|
| **개인 사용** (1~3 채널) | 1~2개 | 4 GB (기본) | 5 |
| **팀 사용** (5~10 채널) | 3~5개 | 8 GB | 3 |
| **헤비 사용** (10~20 채널) | 5~10개 | 16 GB | 3 |
| **최대 규모** | 10~20개 | 24 GB | 2 |

### 현재 환경 권장

yoonhwan 오빠의 M3 Max 36GB 기준:

```
권장 Colima: 8~16 GB (36GB 중 22~44% 할당)
권장 인스턴스: 3~5개
권장 MAX_CONCURRENT: 인스턴스당 3~5
결과: 평상시 3~5개 동시 응답, 최대 15~25개 동시 가능
```

### Colima 설정 변경 방법

```bash
# 현재 설정 확인
colima status

# 확장 (Colima 재시작 필요 — 컨테이너는 유지됨)
colima stop
colima start --cpu 4 --memory 8 --disk 100

# 확인
docker info | grep -E "Total Memory|CPUs"
```

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

## 공유/개별 리소스 분리 (심볼릭 링크)

멀티 인스턴스에서 공유 리소스를 복사하면 드리프트가 발생한다.
심볼릭 링크로 **단일 원본**을 참조하는 구조가 권장된다.

### 컨테이너 마운트 맵

```
Docker 컨테이너 내부                    ← 호스트 소스               공유/개별
─────────────────────────────────────────────────────────────────────────
/workspace/group/          (rw)        ← groups/{name}/            개별
/workspace/global/         (ro)        ← groups/global/            공유
/workspace/project/        (ro)        ← nanoclaw/                 공유 (main만)
/home/node/.claude/        (rw)        ← data/sessions/{name}/    개별
/home/node/.claude/skills/ (rw)        ← container/skills/ (복사)  공유 원본
/workspace/ipc/            (rw)        ← data/ipc/{name}/          개별
```

### 심링크 가능/불가 판정

| 리소스 | 심링크 | 이유 |
|--------|--------|------|
| `groups/global/` | ✅ 가능 | 읽기 전용 마운트. 원본 하나를 모든 인스턴스가 참조 |
| `container/skills/` | ✅ 가능 | 스킬 원본. 컨테이너 시작 시 세션에 복사됨 |
| `data/certs/` | ✅ 가능 | OneCLI CA 인증서. 읽기 전용 |
| `container/agent-runner/` | ⚠️ 불필요 | Docker 이미지에 포함. 이미지 자체가 공유 |
| `groups/{name}/` | ❌ 불가 | 에이전트별 독립 성격/메모리 |
| `store/messages.db` | ❌ 불가 | SQLite 동시 쓰기 불가 |
| `data/sessions/` | ❌ 불가 | 세션 격리 필수 |
| `.env` | ❌ 불가 | 인스턴스별 다른 봇 토큰 |

### 권장 디렉토리 구조 (심링크 적용)

```
~/Project/xclaw/
├── nanoclaw-shared/                     # ★ 공유 리소스 (단일 원본)
│   ├── global/                          #   공유 지침
│   │   └── CLAUDE.md                    #   모든 에이전트가 읽는 공통 규칙
│   ├── skills/                          #   공유 스킬
│   │   ├── agent-browser/
│   │   ├── capabilities/
│   │   ├── slack-formatting/
│   │   └── status/
│   └── certs/                           #   공유 인증서
│       └── onecli-ca.pem
│
├── nanoclaw/                            # 인스턴스 A: Nano
│   ├── groups/
│   │   ├── global → ../../nanoclaw-shared/global       ← 심링크
│   │   └── slack_xclaw/CLAUDE.md                        ← 개별
│   ├── container/
│   │   └── skills → ../../nanoclaw-shared/skills        ← 심링크
│   ├── data/certs → ../../nanoclaw-shared/certs         ← 심링크
│   ├── .env                                              ← 개별
│   └── store/messages.db                                 ← 개별
│
├── nanoclaw-jarvis/                     # 인스턴스 B: Jarvis
│   ├── groups/
│   │   ├── global → ../../nanoclaw-shared/global       ← 심링크
│   │   └── slack_jarvis-dev/CLAUDE.md                   ← 개별
│   ├── container/
│   │   └── skills → ../../nanoclaw-shared/skills        ← 심링크
│   ├── data/certs → ../../nanoclaw-shared/certs         ← 심링크
│   ├── .env                                              ← 개별
│   └── store/messages.db                                 ← 개별
```

### 심링크 셋업 스크립트

```bash
#!/bin/bash
# setup-shared.sh — 공유 리소스 분리 + 심링크 생성
SHARED=~/Project/xclaw/nanoclaw-shared
ORIGIN=~/Project/xclaw/nanoclaw

# 1. 공유 디렉토리 생성 + 원본에서 이동
mkdir -p "$SHARED"
cp -r "$ORIGIN/groups/global" "$SHARED/global"
cp -r "$ORIGIN/container/skills" "$SHARED/skills"
cp -r "$ORIGIN/data/certs" "$SHARED/certs"

# 2. 원본 인스턴스에 심링크 적용
rm -rf "$ORIGIN/groups/global"
ln -sf "$SHARED/global" "$ORIGIN/groups/global"
rm -rf "$ORIGIN/container/skills"
ln -sf "$SHARED/skills" "$ORIGIN/container/skills"
rm -rf "$ORIGIN/data/certs"
ln -sf "$SHARED/certs" "$ORIGIN/data/certs"

# 3. 추가 인스턴스에 심링크 적용
for INST in nanoclaw-jarvis nanoclaw-trader; do
  DIR=~/Project/xclaw/$INST
  [ ! -d "$DIR" ] && continue
  rm -rf "$DIR/groups/global"
  ln -sf "$SHARED/global" "$DIR/groups/global"
  rm -rf "$DIR/container/skills"
  ln -sf "$SHARED/skills" "$DIR/container/skills"
  mkdir -p "$DIR/data"
  rm -rf "$DIR/data/certs"
  ln -sf "$SHARED/certs" "$DIR/data/certs"
  echo "✅ $INST linked"
done
```

### MCP 서버 공유

현재 MCP 설정은 `container/agent-runner/src/index.ts`에 하드코딩되어 있고,
Docker 이미지(`nanoclaw-agent:latest`)에 빌드된다.

**모든 인스턴스가 같은 이미지를 사용하므로 MCP는 자동 공유.**

인스턴스별 다른 MCP가 필요하면:
1. 인스턴스별 다른 이미지 빌드 (비효율)
2. MCP 설정을 외부 JSON으로 분리 + 마운트 (미래 개선)

### 공유 지침 편집 흐름

```
nanoclaw-shared/global/CLAUDE.md 수정
  → 모든 인스턴스가 심링크로 즉시 반영
  → 서비스 재시작 불필요 (다음 컨테이너 스폰 시 자동 적용)
```

개별 지침은 각 `groups/{name}/CLAUDE.md`에서 독립 편집.

## 주의사항

1. **컨테이너 이름 충돌**: 각 인스턴스가 `nanoclaw-slack-{group}-{ts}` 패턴으로 컨테이너 생성. 그룹명이 다르면 충돌 없음
2. **OneCLI 포트**: 모든 인스턴스가 같은 OneCLI(10254/10255) 공유. 에이전트별 rate limit 필요 시 OneCLI 대시보드에서 설정
3. **동시 컨테이너**: 기본 `MAX_CONCURRENT_CONTAINERS=5`가 인스턴스별 적용. 3개 인스턴스 × 5 = 최대 15개 동시 컨테이너 가능
4. **같은 채널에 여러 봇**: 가능하지만 권장하지 않음. 메시지 중복 처리됨
5. **Slack 봇 이름/아바타**: api.slack.com/apps → 각 앱의 Display Information에서 설정
