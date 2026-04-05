# NanoClaw 완전 가이드 🤖

**Slack/Telegram에서 개인 AI 에이전트를 운영하는 완전 초보자 가이드**

> 개발 경험이 없어도 괜찮습니다. 이 가이드는 클릭 하나하나를 함께 진행합니다.

---

## 목차

1. [NanoClaw가 뭔가요?](#nanoclaw가-뭔가요)
2. [아키텍처 (간단히)](#아키텍처-간단히)
3. [비용이 얼마인가요?](#비용이-얼마인가요)
4. [사전 준비물 체크리스트](#사전-준비물-체크리스트)
5. [설치 과정 (단계별)](#설치-과정-단계별)
6. [Slack 앱 설정하기](#slack-앱-설정하기)
7. [채널 추가 및 테스트](#채널-추가-및-테스트)
8. [일상 관리](#일상-관리)
9. [트러블슈팅](#트러블슈팅)
10. [자주 묻는 질문](#자주-묻는-질문)

---

## NanoClaw가 뭔가요?

NanoClaw는 **Slack, Telegram, Discord 등 메신저에서 바로 사용할 수 있는 개인 AI 에이전트**입니다.

### 간단한 예시
```
당신: @Andy 다음주 일정 요약해줄래?
(NanoClaw가 처리 중...)
Andy: 월요일: 팀 미팅 10시, 수요일: 프레젠테이션...
```

### NanoClaw의 특징

✅ **강력함**: Claude Code와 동일한 Anthropic AI 엔진 사용  
✅ **안전함**: 각 채널이 독립된 Docker 컨테이너에서 실행 → 파일 격리, 권한 제어  
✅ **간단함**: 설정이 거의 없고, Claude Code가 대부분 자동으로 처리  
✅ **커스터마이징 가능**: 당신의 필요에 맞게 수정 가능  
✅ **멀티채널**: Slack + Telegram + Discord 동시 운영 가능  

---

## 아키텍처 (간단히)

### 메시지 흐름

```
당신의 메시지 (Slack)
    ↓
NanoClaw (Node.js 프로세스)
    ↓
Docker 컨테이너 (격리된 실행 환경)
    ↓
Claude Agent SDK (Anthropic 공식 AI)
    ↓
OneCLI (API 키 안전 관리)
    ↓
Anthropic API (Claude 모델)
    ↓
응답 → 당신에게 전달
```

### 핵심 개념

| 용어 | 설명 |
|------|------|
| **NanoClaw** | Mac에서 실행되는 메인 프로세스 (Node.js) |
| **Docker** | 컨테이너 엔진 - 격리된 환경에서 AI 실행 |
| **Claude Agent SDK** | Anthropic이 만든 공식 AI 프레임워크 |
| **OneCLI** | API 키와 토큰을 안전하게 관리하는 프록시 |
| **채널** | Slack, Telegram, Discord 등 메신저 앱 |

**보안 포인트:**
- 각 채널은 독립된 컨테이너에서 실행
- API 키는 컨테이너에 전달되지 않음 (OneCLI를 통해 주입)
- 파일 접근은 마운트된 폴더만 가능
- Bash 명령도 컨테이너 내부에서 안전하게 실행

---

## 비용이 얼마인가요?

### 옵션 1: Claude 구독 (권장)
- **Claude Pro**: $20/월 (표준 사용자)
- **Claude Max**: $100/월 (고용량 사용자)
- NanoClaw 자체: 무료 (오픈소스)
- **추가 비용**: 없음

### 옵션 2: Anthropic API (종량제)
- 사용한 만큼만 지불
- Claude 구독이 없는 사람도 가능
- 가격: [Anthropic 가격 페이지](https://www.anthropic.com/pricing) 참조

### 옵션 3: 로컬 모델
- 무료 (하지만 성능이 떨어짐)
- Ollama 같은 도구로 로컬 모델 실행 가능

**저는 뭘 선택해야 하나요?**
- 매일 사용할 거면: **Claude Pro** ($20/월 가장 저렴)
- 가끔만 사용할 거면: **API (종량제)**
- 극도로 절약하려면: **로컬 모델 (올라마)**

---

## 사전 준비물 체크리스트

설치하기 전에 아래를 확인해주세요.

### 필수 준비물
- [ ] **macOS** (Apple Silicon 또는 Intel, Big Sur 이상)
- [ ] **Homebrew** (Mac용 패키지 관리자)
- [ ] **GitHub 계정** (NanoClaw 포크할 때 필요)
- [ ] **Claude Pro/Max** 또는 **Anthropic API 키**
- [ ] **Slack 워크스페이스** (또는 Telegram 계정)

### 준비물 확인하기

**1. macOS 버전 확인**
```bash
sw_vers
```
"ProductVersion"이 11.0 이상인지 확인

**2. Homebrew 설치 확인**
```bash
brew --version
```
설치 안 됐으면:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

**3. GitHub 계정**
- [github.com](https://github.com)에서 가입 (5분)

**4. Claude 구독**
- [claude.ai](https://claude.ai)에 로그인 → 왼쪽 상단에서 구독 선택

---

## 설치 과정 (단계별)

### 📋 전체 흐름

| 단계 | 시간 | 내용 |
|------|------|------|
| 1 | 5분 | 필수 도구 설치 |
| 2 | 5분 | GitHub Fork & Clone |
| 3 | 10분 | Claude Code 설치 & 토큰 |
| 4 | 15분 | NanoClaw 자동 설정 |
| 5 | 10분 | Slack 앱 생성 |
| 6 | 5분 | 테스트 메시지 보내기 |
| **총** | **50분** | |

---

### Step 1️⃣: 필수 도구 설치 (5분)

터미널을 열고 아래 명령어를 하나씩 실행하세요.

**1-1. Node.js 설치**
```bash
brew install node@22
```

**1-2. Git 설치**
```bash
brew install git
```

**1-3. GitHub CLI 설치**
```bash
brew install gh
```

**1-4. Docker 설치**

두 가지 옵션 중 하나 선택:

**옵션 A: Docker Desktop (권장, 더 간단)**
```bash
brew install --cask docker
```
설치 완료 후 Spotlight (Command+Space)에서 "Docker" 검색 후 실행

**옵션 B: Colima (더 가벼움)**
```bash
brew install colima docker docker-compose
colima start
```

---

### Step 2️⃣: GitHub Fork & Clone (5분)

**2-1. GitHub에 로그인**
```bash
gh auth login
```
화면에 따라 진행하면 됩니다.

**2-2. NanoClaw Fork & Clone**
```bash
gh repo fork qwibitai/nanoclaw --clone
```

**2-3. 폴더로 이동**
```bash
cd nanoclaw
```

---

### Step 3️⃣: Claude Code 설치 & 토큰 (10분)

**3-1. Claude Code 설치** (Mac에서)
```bash
npm install -g @anthropic-ai/claude-code
```

**3-2. Claude Code 최초 실행**
```bash
claude
```
화면에 "Welcome to Claude Code" 메시지가 나오면, 당신의 Claude 계정으로 로그인하세요.

**3-3. 토큰 발급 및 복사**
```bash
claude setup-token
```
출력된 토큰을 **메모장에 복사해두세요.** (나중에 필요)

예시:
```
Your token: claude_token_sk-xxxxxxxxxxxx...
```

---

### Step 4️⃣: NanoClaw 자동 설정 (15분)

**4-1. Claude Code 실행**
```bash
claude
```

**4-2. /setup 명령 실행**
Claude Code 프롬프트에서:
```
/setup
```

**4-3. 대화형 설정 진행**

Claude가 물어보는 질문들에 답하세요:

| 질문 | 답변 |
|------|------|
| Docker 또는 Apple Container? | **Docker** 선택 |
| Claude 구독 또는 API 키? | **Claude 구독** 선택 (또는 API 키) |
| 토큰 입력 | Step 3에서 복사한 토큰 붙여넣기 |
| 채널 선택 | **Slack** 선택 (나중에 Telegram 추가 가능) |
| 에이전트 이름 | 원하는 이름 (예: Andy, Claude, Alex) |

✅ **설정 완료 메시지가 나오면 성공!**

---

### Step 5️⃣: Slack 앱 설정 (10분)

NanoClaw가 Slack과 연결되려면 Slack 앱을 만들어야 합니다.

#### 5-1. Slack 앱 생성

1. [api.slack.com/apps](https://api.slack.com/apps) 열기
2. **"Create New App"** 클릭
3. **"From scratch"** 선택
4. **App Name**: `NanoClaw` (또는 원하는 이름)
5. **Select a workspace**: 당신의 Slack 워크스페이스 선택
6. **"Create App"** 클릭

#### 5-2. Socket Mode 활성화

1. 왼쪽 메뉴에서 **Socket Mode** 클릭
2. **"Enable Socket Mode"** 토글 켜기
3. **App-Level Token** 섹션에서 **"Generate"** 클릭
4. Token Name: `NanoClaw` 입력
5. **"Generate"** 클릭
6. 토큰 복사 (👉 **`xapp-...`**로 시작)
7. **메모장에 저장**

#### 5-3. 이벤트 구독

1. 왼쪽 메뉴에서 **Event Subscriptions** 클릭
2. 토글을 **"On"** 으로 켜기
3. **"Subscribe to bot events"** 섹션 아래로 스크롤
4. 아래 이벤트들 추가:
   - `message.channels`
   - `message.groups`
   - `message.im`
5. **"Save Changes"** 클릭

#### 5-4. OAuth Scopes 설정

1. 왼쪽 메뉴에서 **OAuth & Permissions** 클릭
2. **"Scopes"** → **"Bot Token Scopes"** 섹션으로 이동
3. **"Add an OAuth Scope"** 버튼 클릭
4. 아래 스코프들을 하나씩 추가:
   - `chat:write`
   - `channels:history`
   - `groups:history`
   - `im:history`
   - `channels:read`
   - `groups:read`
   - `users:read`
5. 페이지 위로 스크롤해서 **"Install to Workspace"** 클릭

#### 5-5. Bot Token 복사

1. 설치 완료 후 **"Bot User OAuth Token"** 섹션 찾기
2. 토큰 복사 (👉 **`xoxb-...`**로 시작)
3. **메모장에 저장**

#### 5-6. NanoClaw에 토큰 추가

터미널에서:
```bash
cat >> .env << EOF
SLACK_BOT_TOKEN=xoxb-복사한_토큰_여기
SLACK_APP_TOKEN=xapp-복사한_토큰_여기
EOF
```

#### 5-7. NanoClaw 재시작

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

---

### Step 6️⃣: 테스트 메시지 보내기 (5분)

#### 6-1. Slack에서 채널 추가

1. Slack 열기
2. 원하는 채널 선택 (또는 새로 만들기)
3. 채널 이름 옆 **▼** 클릭 → **"Integrations"** → **"Add apps"**
4. 검색에서 `NanoClaw` 찾기
5. **"Add"** 클릭

#### 6-2. 메시지 보내기

채널에서:
```
@NanoClaw 안녕! 너 뭐할 수 있어?
```

**잠깐 기다리면 (5-10초) 응답이 옵니다!**

예상 응답:
```
안녕하세요! 저는 NanoClaw예요. 저는 다음을 할 수 있습니다:
- 질문에 답변
- 파일 분석
- 웹 검색
- 코드 작성 및 리뷰
...
```

✅ **응답이 오면 설치 완료!**

---

## Slack 앱 설정하기 (상세)

위의 Step 5를 자세히 설명합니다.

### 토큰이란?

Slack 앱이 Slack과 통신할 때 필요한 암호 같은 것입니다.

- **Bot Token** (`xoxb-...`): 메시지 보내기, 읽기 권한
- **App Token** (`xapp-...`): Socket Mode 연결 권한

### Slack 앱 생성 체크리스트

- [ ] [api.slack.com/apps](https://api.slack.com/apps)에서 앱 생성
- [ ] App Name: `NanoClaw` 입력
- [ ] Workspace 선택
- [ ] Socket Mode 활성화
- [ ] App-Level Token 생성 (`xapp-...` 저장)
- [ ] Event Subscriptions 활성화
- [ ] 3개 이벤트 구독 (`message.channels`, `message.groups`, `message.im`)
- [ ] 7개 OAuth Scope 추가
- [ ] Workspace에 설치
- [ ] Bot Token 복사 (`xoxb-...` 저장)
- [ ] .env 파일에 두 토큰 추가
- [ ] NanoClaw 재시작
- [ ] Slack 채널에 봇 추가
- [ ] 테스트 메시지 전송

---

## 채널 추가 및 테스트

Slack을 넘어 다른 채널도 추가할 수 있습니다.

### Telegram 추가하기 (선택사항)

```bash
claude
```

Claude Code 프롬프트에서:
```
/add-telegram
```

Claude가 대화형으로 가이드합니다. Telegram Bot Token 발급받고 입력하면 됩니다.

### Discord 추가하기 (선택사항)

```bash
/add-discord
```

### Gmail 추가하기 (선택사항)

```bash
/add-gmail
```

---

## 일상 관리

### 서비스 상태 확인

**실행 중인가?**
```bash
launchctl list | grep nanoclaw
```

출력에 `com.nanoclaw`가 있으면 실행 중입니다.

### 서비스 관리

**재시작하기** (설정 변경 후)
```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

**중지하기**
```bash
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```

**시작하기**
```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
```

### 로그 확인

**메인 로그**
```bash
tail -f logs/nanoclaw.log
```

**에러만 보기**
```bash
tail -f logs/nanoclaw.error.log
```

**최근 100줄**
```bash
tail -100 logs/nanoclaw.log
```

### 커스터마이징

#### 에이전트 이름 바꾸기

```bash
claude
```

그 다음:
```
/customize
```

Claude가 대화형으로 도와줍니다.

#### 에이전트 성격 설정

각 채널 폴더의 `CLAUDE.md` 파일 수정:

```bash
nano groups/slack_main/CLAUDE.md
```

예시:
```markdown
# NanoClaw 성격

당신은 친절하고 간결한 AI 어시스턴트입니다.
- 항상 존댓말 사용
- 최대 2-3문장으로 답변
- 이모지 적게 사용
```

### 업데이트하기

```bash
claude
```

그 다음:
```
/update-nanoclaw
```

Claude가 최신 버전으로 업데이트합니다.

---

## 트러블슈팅

### 문제: "Docker is not running"

**해결:**
```bash
# Docker Desktop 사용 시
open -a Docker

# Colima 사용 시
colima start
```

### 문제: 봇이 메시지를 받지 못함

**체크리스트:**
1. 봇이 Slack 채널에 추가되었는가?
   - 채널 설정 → Integrations → Add apps → NanoClaw
2. NanoClaw 서비스가 실행 중인가?
   ```bash
   launchctl list | grep nanoclaw
   ```
3. 토큰이 올바른가?
   ```bash
   cat .env | grep SLACK
   ```
4. 로그에 에러가 있는가?
   ```bash
   tail -50 logs/nanoclaw.error.log
   ```

### 문제: "Self-signed certificate detected"

이는 OneCLI 인증서 문제입니다.

**해결:**
```bash
mkdir -p data/certs
curl -s http://127.0.0.1:10254/api/gateway/ca > data/certs/onecli-ca.pem
./container/build.sh
```

### 문제: "Extra usage is required"

이는 Claude 모델의 context window가 가득 찬 경우입니다.

**해결:**
1. `container/agent-runner/src/index.ts` 파일 열기
2. 모델을 `claude-3-5-sonnet` 또는 `claude-3-5-haiku`로 변경
3. 재빌드:
   ```bash
   ./container/build.sh
   ```

또는 Claude Max로 업그레이드하기.

### 문제: OneCLI postgres 포트 충돌

이는 당신의 Mac에 이미 PostgreSQL이 실행 중인 경우입니다.

**해결:**
```bash
nano ~/.onecli/.env
```

아래 줄 추가:
```
POSTGRES_PORT=5434
```

### 문제: 권한 오류 (Permission denied)

```bash
chmod +x container/build.sh
chmod +x scripts/*.sh
```

---

## 자주 묻는 질문

### Q1: Claude Code와 뭐가 다른가요?

**A:**
- **Claude Code**: 터미널에서 대화
- **NanoClaw**: Slack, Telegram 등 메신저에서 대화

같은 Claude AI를 사용하지만, 접근 방식이 다릅니다.

### Q2: 비용이 정말 안 드나요?

**A:** 맞습니다.
- Claude Pro ($20/월) 구독만으로 충분
- NanoClaw 자체는 무료 (오픈소스)
- Docker는 무료
- 추가 비용 없음

### Q3: 여러 채널을 동시에 쓸 수 있나요?

**A:** 네, 가능합니다.
- Slack + Telegram + Discord 동시 운영 가능
- 각 채널이 독립적으로 작동
- `/add-telegram`, `/add-discord` 등으로 추가

### Q4: 에이전트가 내 파일을 볼 수 있나요?

**A:** 마운트된 폴더만 가능합니다.

NanoClaw는 컨테이너 격리를 사용하므로:
- 기본적으로 `groups/` 폴더만 접근 가능
- 다른 폴더는 명시적으로 추가할 수 있음
- 권한은 철저히 제어됨

### Q5: Mac을 꺼도 계속 돌아가나요?

**A:** 아니요, Mac이 켜져 있어야 합니다.

NanoClaw는 Mac의 launchd 서비스로 등록되어:
- Mac이 켜진 동안 자동으로 실행
- Mac을 끄면 정지됨
- 서버로 옮기려면 Linux 버전 설치 필요

### Q6: VPN 사용해도 되나요?

**A:** 네, 문제없습니다.

다만:
- API 호출은 VPN을 통해 갑니다
- 속도가 약간 느릴 수 있음
- 보안은 더 좋아집니다

### Q7: 오프라인으로 쓸 수 있나요?

**A:** 아니요, 인터넷이 필수입니다.

- Claude API 호출이 필요
- 로컬 모델(Ollama) 사용 가능하지만 성능 낮음

### Q8: 무언가 잘못됐을 때는?

**A:** Claude Code에 물어보세요!

```bash
claude
```

그 다음:
```
/debug
```

Claude가 자동으로 진단하고 수정해줍니다.

### Q9: 내가 만든 수정사항을 어디다 저장하나요?

**A:** Git 브랜치에 저장합니다.

당신의 fork는 이미 GitHub에 있으므로:
```bash
git status
git add .
git commit -m "내 커스터마이징"
git push
```

### Q10: 다시 설치하려면?

**A:** 간단합니다.

```bash
cd ..
rm -rf nanoclaw
gh repo fork qwibitai/nanoclaw --clone
cd nanoclaw
claude
/setup
```

---

## 다음 단계

축하합니다! NanoClaw가 설치되었습니다. 🎉

### 이제 할 수 있는 것들

1. **일상 자동화**
   ```
   @NanoClaw 매일 아침 9시에 날씨 알려줄래?
   ```

2. **분석 및 요약**
   ```
   @NanoClaw 이 문서 요약해줄래?
   ```

3. **코드 리뷰** (개발자라면)
   ```
   @NanoClaw 이 코드 리뷰해줄래?
   ```

4. **창의적인 일**
   ```
   @NanoClaw 블로그 포스트 아이디어 10개 줄래?
   ```

### 더 알아보기

- 공식 문서: [docs.nanoclaw.dev](https://docs.nanoclaw.dev)
- 커뮤니티 Discord: [discord.gg/VDdww8qS42](https://discord.gg/VDdww8qS42)
- GitHub: [github.com/qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw)

---

## 도움말

설치 중 막히는 부분이 있으면:

```bash
claude
```

그 다음:
```
"[당신의 문제 설명]"
```

예시:
```
"Docker가 실행되지 않아요"
"Slack 토큰을 어디서 찾아야 하는지 모르겠어요"
"메시지 응답이 안 와요"
```

Claude가 대화형으로 해결해줄 것입니다. 🚀

---

---

## OpenClaw에서 마이그레이션

기존에 OpenClaw(자비스 등)을 사용하고 있다면, NanoClaw로 마이그레이션할 수 있습니다.

### 마이그레이션이란?

OpenClaw의 설정, 채널 인증정보, 예약 작업 등을 NanoClaw으로 가져오는 과정입니다.

| OpenClaw | → | NanoClaw |
|----------|---|----------|
| `openclaw.json` 설정 | → | `.env` + CLAUDE.md |
| Slack bot/app 토큰 | → | `.env` (동일 토큰 재사용 또는 새 봇 생성) |
| SOUL.md / IDENTITY.md | → | `groups/global/CLAUDE.md` 또는 그룹별 CLAUDE.md |
| USER.md / MEMORY.md | → | `user-context.md` / `memories.md` |
| 크론잡 | → | NanoClaw 스케줄러 (DB) |
| 워크스페이스 스킬 | → | `container/skills/` (직접 복사 가능) |

### 마이그레이션 방법

**셋업 중 자동 감지:**
`/setup` 실행 시 `~/.openclaw` 폴더가 감지되면 마이그레이션 옵션이 자동으로 제시됩니다.

**수동 실행:**
```
/migrate-from-openclaw
```

### 마이그레이션 단계

1. **Discovery** — OpenClaw 설치 스캔 (채널, 그룹, 스킬, 크론잡 등)
2. **그룹 설계** — 공유 성격 vs 완전 분리 vs 메인만 선택
3. **설정 추출** — 타임존, Slack 인증정보, 허용목록
4. **아이덴티티** — SOUL.md/IDENTITY.md → CLAUDE.md 통합 (선택)
5. **채널 인증정보** — Slack 토큰을 `.env`에 저장
6. **예약 작업** — 크론잡 이전 (선택)
7. **완료 요약**

### 주의사항

- **같은 Slack 봇 토큰을 공유하면**, OpenClaw과 NanoClaw이 같은 메시지에 동시 응답할 수 있습니다
- **권장**: NanoClaw 전용 Slack 앱을 새로 만드세요 ([Slack 앱 설정하기](#slack-앱-설정하기) 참고)
- **WhatsApp**: 인증 세션이 복사되지 않습니다. NanoClaw에서 새로 인증 필요
- **Slack 채널 ID는 대문자**여야 합니다 (예: `C0AQS1H917X`). 소문자로 등록되면 메시지를 못 받음

### 기존 에이전트(자비스 등) 완전 교체

OpenClaw을 완전히 NanoClaw로 교체하려면:

1. `/migrate-from-openclaw`로 모든 데이터 이전
2. NanoClaw 전용 Slack 앱 생성 (새 봇)
3. 기존 채널들에서 OpenClaw 봇 제거 → NanoClaw 봇 추가
4. OpenClaw 서비스 중지:
   ```bash
   launchctl unload ~/Library/LaunchAgents/ai.openclaw.gateway.plist
   ```
5. NanoClaw 정상 작동 확인 후, OpenClaw은 백업으로 유지하거나 삭제

---

**마지막 팁**: 당신의 fork를 북마크해두세요. NanoClaw 커스터마이징을 저장할 때 자주 돌아올 거예요!
