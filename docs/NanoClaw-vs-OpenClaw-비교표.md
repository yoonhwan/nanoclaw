# NanoClaw vs OpenClaw 비교표

## 핵심 아키텍처

| 항목 | OpenClaw | NanoClaw |
|------|----------|----------|
| **실행 모델** | 싱글 프로세스 (모든 그룹 공유) | 그룹별 독립 Docker 컨테이너 |
| **AI 엔진** | Anthropic API 직접 호출 | Claude Agent SDK (Claude Code 동일 엔진) |
| **도구 사용** | 자체 도구 시스템 | Claude Code 도구 (Bash, Read, Write, Edit, Grep, Glob) |
| **격리** | 없음 (같은 프로세스) | 컨테이너 격리 (파일시스템, 네트워크 독립) |
| **크레덴셜** | openclaw.json에 직접 저장 | OneCLI Agent Vault (프록시 주입, 컨테이너에 시크릿 없음) |
| **설정** | openclaw.json (단일 설정 파일) | .env + CLAUDE.md (코드 기반) |
| **메모리** | SOUL.md + MEMORY.md + 일일 메모리 | 그룹별 CLAUDE.md + global/CLAUDE.md |

## 채널 지원

| 채널 | OpenClaw | NanoClaw |
|------|----------|----------|
| **Slack** | ✅ | ✅ |
| **WhatsApp** | ✅ | ✅ (`/add-whatsapp`) |
| **Telegram** | ❌ | ✅ (`/add-telegram`) |
| **Discord** | ❌ | ✅ (`/add-discord`) |
| **Gmail** | ❌ | ✅ (`/add-gmail`) |
| **iMessage** | ✅ | ❌ |
| **Signal** | ✅ | ❌ (계획 중) |
| **Matrix** | ✅ | ❌ |
| **Emacs** | ❌ | ✅ (`/add-emacs`) |
| **CLI (터미널)** | ❌ | ✅ (`/claw`) |
| **멀티채널 동시** | ✅ | ✅ |

## 기능 비교

### ✅ 둘 다 제공

| 기능 | OpenClaw | NanoClaw |
|------|----------|----------|
| 메시지 응답 | ✅ | ✅ |
| 예약 작업 (크론) | ✅ (28개 지원) | ✅ (DB 기반 스케줄러) |
| 웹 검색 | ✅ (Perplexity 내장) | ✅ (MCP 서버로 확장) |
| 파일 실행 | ✅ (exec: full) | ✅ (컨테이너 안에서 자유) |
| 스킬 시스템 | ✅ (워크스페이스 스킬) | ✅ (스킬 브랜치 머지) |
| 트리거 패턴 | ✅ (@멘션) | ✅ (@멘션, 메인은 트리거 불필요) |
| 허용 발신자 | ✅ (allowFrom) | ✅ (sender-allowlist.json) |
| 서비스 자동시작 | ✅ (launchd) | ✅ (launchd / systemd) |

### 🟢 NanoClaw만 제공

| 기능 | 설명 |
|------|------|
| **컨테이너 격리** | 각 그룹이 독립 Docker 컨테이너. 파일시스템 완전 분리 |
| **Claude Code 도구** | Bash, Read, Write, Edit 등 Claude Code와 동일한 도구셋 |
| **MCP 서버** | 컨테이너 안에서 MCP 서버 실행 가능 (확장성 무한) |
| **이미지 인식** | `/add-image-vision` — 이미지 첨부 분석 |
| **PDF 읽기** | `/add-pdf-reader` — PDF 텍스트 추출 |
| **음성 전사** | `/add-voice-transcription` — 음성 메시지 자동 전사 |
| **텔레그램 스웜** | `/add-telegram-swarm` — 여러 봇이 팀으로 동작 |
| **위키** | `/add-karpathy-llm-wiki` — 지속적 지식 베이스 |
| **macOS 상태바** | `/add-macos-statusbar` — 메뉴바 아이콘 |
| **Ollama 로컬 모델** | `/add-ollama-tool` — 로컬 LLM 연동 |
| **마운트 허용목록** | 에이전트가 접근할 디렉토리 명시적 제어 |
| **그룹별 성격** | 각 그룹마다 독립된 CLAUDE.md (다른 성격 가능) |
| **OneCLI 보안** | API 키가 컨테이너에 노출 안 됨 |
| **GitHub Fork** | 개인 커스터마이징을 Git으로 관리 |
| **채널 포맷팅** | `/channel-formatting` — 채널별 마크다운 변환 |
| **CLI 도구** | `/claw` — 채팅앱 없이 터미널에서 에이전트 실행 |

### 🔴 OpenClaw만 제공 (NanoClaw에 없음)

| 기능 | 설명 | 대안 |
|------|------|------|
| **TTS (음성 합성)** | ElevenLabs 연동, 음성 응답 | MCP 서버로 직접 구현 가능 |
| **iMessage** | Apple iMessage 채널 | 미지원 (Apple 제한) |
| **Signal** | Signal 메신저 | 미지원 (계획 중) |
| **Matrix** | Matrix 프로토콜 | 미지원 |
| **공유 워크스페이스** | 모든 그룹이 같은 메모리/성격 | `groups/global/CLAUDE.md`로 부분 공유 |
| **일일 메모리 자동 기록** | 매일 자동 메모리 정리 | 크론잡으로 구현 가능 |
| **Human delay** | 사람처럼 타이핑 지연 | 미지원 (불필요) |
| **Exec approval** | 명령어 실행 승인 UI | 컨테이너 격리로 대체 |
| **멀티 모델 프로바이더** | Google, OpenAI 등 동시 | Anthropic 전용 (MCP로 확장 가능) |
| **1Password 연동** | 비밀번호 관리자 통합 | OneCLI vault 사용 |
| **Perplexity 내장** | 웹 검색 내장 | MCP 서버로 추가 |

## 비용 비교

| 항목 | OpenClaw | NanoClaw |
|------|----------|----------|
| **소프트웨어** | 무료 (오픈소스) | 무료 (오픈소스) |
| **AI 모델** | Anthropic API (종량제) 또는 구독 | Anthropic API 또는 Claude Pro/Max 구독 |
| **인프라** | 없음 (로컬 프로세스) | Docker 무료 (Colima) |
| **추가 서비스** | Perplexity, ElevenLabs 등 별도 | 필요시 MCP로 추가 |

## 마이그레이션 난이도

| 항목 | 난이도 | 방법 |
|------|--------|------|
| Slack 인증정보 | ⭐ 쉬움 | 토큰 복사 또는 새 앱 생성 |
| 성격/아이덴티티 | ⭐⭐ 보통 | SOUL.md → CLAUDE.md 수동 편집 |
| 메모리 | ⭐⭐ 보통 | MEMORY.md → memories.md 복사 |
| 크론잡 | ⭐⭐⭐ 약간 복잡 | `/migrate-from-openclaw` 자동 |
| 워크스페이스 스킬 | ⭐ 쉬움 | `container/skills/`에 복사 |
| 전체 자동 마이그레이션 | ⭐⭐ 보통 | `/migrate-from-openclaw` 실행 |

## 결론

| 선택 기준 | OpenClaw | NanoClaw |
|-----------|----------|----------|
| **보안이 중요** | | ✅ (컨테이너 격리 + OneCLI) |
| **채널 다양성** | | ✅ (Telegram, Discord, Gmail) |
| **Claude Code 도구** | | ✅ (동일 엔진) |
| **iMessage 필수** | ✅ | |
| **TTS 필수** | ✅ | (MCP로 가능) |
| **멀티 모델 필요** | ✅ | (Anthropic 전용) |
| **간편한 설정** | ✅ (단일 JSON) | (코드 기반) |
| **확장성** | | ✅ (MCP + 스킬 시스템) |
| **비개발자 접근성** | ✅ (약간 쉬움) | ⭐ (가이드 따라하면 가능) |
