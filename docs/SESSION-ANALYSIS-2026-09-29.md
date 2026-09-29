# brew-browser 전수조사 & 활용 전략 분석

> **작성일**: 2026-09-29
> **작성**: Claude Code (Karina 페르소나) — `CLAUDE.md` 워크플로우 준수
> **대상 레포**: 이 레포(포크) + 원본
> **세션 목적**: 레포 전수조사 → 정체 파악 → 활용/수익화 전략 도출

---

## 🔗 GitHub 주소

| 구분 | URL |
|---|---|
| **이 레포 (포크)** | https://github.com/bmshin94/brew-browser |
| **원본 (upstream)** | https://github.com/msitarzewski/brew-browser |
| 릴리스 (다운로드) | https://github.com/msitarzewski/brew-browser/releases/latest |
| Linux CI 빌드 | https://github.com/msitarzewski/brew-browser/actions/workflows/linux-build.yml |
| 후원 (원저작자) | https://github.com/sponsors/msitarzewski |
| 랜딩 페이지 | https://brew-browser.zerologic.com |
| 이 브랜치 | `claude/kind-bell-lgq2c0` |

**포크 상태**: 원본을 포크한 뒤 `CLAUDE.md`(AGENTS.md 워크플로우 + Karina 페르소나)만 추가된 상태
(`f306623 Add files via upload`, `8db4a81 Update CLAUDE.md`)

---

## 1. 이게 뭐하는 프로젝트인가

### 한 줄 정의

**macOS / Linux용 Homebrew GUI 데스크톱 애플리케이션.** 터미널에서 `brew install ...`
치던 작업을 GUI 클릭으로 수행. 단, 모든 동작은 실제 `brew` CLI로 shell out 하며
출력 전체가 Activity 드로어에 실시간 스트리밍됨.

| 항목 | 값 |
|---|---|
| 원저작자 | Michael Sitarzewski (`package.json:7`) |
| 버전 | 0.7.2 |
| 라이선스 | MIT (No CLA / No EULA) |
| 식별자 | `com.zerologic.brew-browser` |
| 정체 | **독립 실행 데스크톱 앱** (플러그인/스킬/MCP 아님) |

### 규모 (실측)

| 영역 | 라인 수 |
|---|---|
| Rust (`src-tauri/src/`) | 26,782 |
| Swift (`native/`) | 18,481 |
| Svelte (`src/`, 44개 컴포넌트) | 17,698 |
| TypeScript (`src/`) | 7,649 |
| Memory Bank (`memory-bank/**.md`) | 13,673 |
| **합계** | **약 84,000줄** |

등록된 Tauri 커맨드: **85개** (`src-tauri/src/lib.rs:166` invoke_handler)

---

## 2. 아키텍처 — "앱이 두 개"

`README.md` "Two builds" 섹션. 같은 앱을 두 기술로 각각 완전 구현하고
기능 + 데이터 계약(data contract) 패리티를 유지.

| | **Tauri 빌드** | **Native 빌드** |
|---|---|---|
| 스택 | Tauri 2 · SvelteKit(Svelte 5 runes) · Rust | Swift 6 · SwiftUI · Liquid Glass |
| 대상 | macOS 13+ **및 Linux** | macOS 26 (Tahoe) 전용 |
| 역할 | 실제 배포되는 크로스플랫폼 앱 | "진짜 네이티브" 플래그십, 메모리 약 절반 |
| 소스 | `src/` + `src-tauri/` | `native/` (Swift Package, `.xcodeproj` 없음) |
| 업데이트 | 자체 업데이터 (minisign) | Sparkle 2 (ed25519) |
| 상태 | shipping, 서명 + 노타라이즈 | shipping, 서명 + 노타라이즈 |

**왜 두 개?** SwiftUI는 Linux 미지원, Liquid Glass는 macOS 26 전용.
공유 규약: 동일 `settings.json` 스키마, 동일 `categories.json` / `enrichment.json`,
동일 `brew` / `brew vulns` 호출, 동일 trending/enrichment 엔드포인트.
근거: `memory-bank/decisions.md` 2026-06-01 "keep both" ADR.

### 3층 구조 (Tauri 빌드)

```
1층 화면 (Svelte)  src/          ← fetch() 사용 금지. UI만
        │ invoke("command_name", {...})   ← 유일한 통로, 85개 열거된 커맨드만
2층 두뇌 (Rust)    src-tauri/    ← 검증 + tokio::process 실행 + 스트리밍
        │ 타입드 enum → 인수 배열 (셸 미경유)
3층 실제 brew CLI                ← 시스템에 이미 설치된 것
```

핵심: `tauri-plugin-shell` **미사용** → 임의 명령 실행이 구조적으로 불가능.

---

## 3. 폴더 전수조사

```
brew-browser/
├── src/                        ① Svelte 프론트엔드
│   ├── components/  (44개)     Dashboard, Library, Discover, Bundles, Trending,
│   │                           Snapshots, Services, Settings, CommandPalette,
│   │                           PackageDetail, ActivityDrawer, DeviceFlowModal...
│   ├── stores/      (23개)     Svelte 5 runes 상태관리
│   ├── util/                   readiness, liveBundles, recovery, format, token...
│   ├── api.ts                  ★ Rust 백엔드 타입 안전 invoke() 래퍼 전부
│   ├── types.ts                프론트-백엔드 공용 타입
│   └── styles/                 tokens.css, typography.css, reset.css
│
├── src-tauri/                  ② Rust 백엔드
│   ├── src/commands/ (21모듈)  list search info actions services brewfile
│   │                           trending vulns github catalog updater bundles
│   │                           categories enrichment disk_usage settings env
│   │                           brew_env cask_icon cask_icon_homepage
│   ├── src/brew/               ★ exec.rs(프로세스 실행) parse.rs(출력 파싱)
│   │                           paths.rs(brew --prefix 자동탐지) error_patterns.rs
│   ├── src/vulns/              CVE: client cache enrich fingerprint
│   ├── src/github/             auth(Device Flow) stats url(엄격 검증) actions
│   ├── src/trending/           client cache velocity + history/
│   ├── src/system/profile.rs   RAM / arch / GPU / 디스크 감지
│   ├── src/catalog/            16,000+ 패키지 카탈로그
│   ├── data/                   catalog/*.json.gz (빌드타임 내장, ~6MiB gz)
│   │                           bundles.json categories.json enrichment.json.gz
│   ├── tests/fixtures/ (10개)  실제 brew 출력 샘플 (파서 회귀 테스트)
│   ├── tauri.conf.json         CSP, Developer ID 서명, 업데이터 공개키
│   ├── capabilities/default.json  ★ 권한 화이트리스트 9개만
│   ├── deny.toml / .cargo/audit.toml
│   └── Cargo.toml              tauri2, tokio, reqwest(rustls), sha2, uuid, chrono...
│
├── native/                     ③ Swift/SwiftUI 앱 (별도 구현체)
│   ├── Sources/BrewBrowserKit/ (40개 .swift) 뷰 + Sendable struct 서비스
│   ├── Sources/BrewBrowser/    앱 엔트리
│   ├── Tests/ (16개)           BrewOutputParsing, VulnsParsing, BundleReadiness,
│   │                           SettingsContract, RosettaDetection...
│   └── Package.swift
│
├── recipes/                    ④ 번들 레시피 9개 + JSON Schema
│   local-llm image-gen graphics media web-dev databases
│   agentic-web-dev lamp lemp + recipe.schema.json
│
├── memory-bank/                ⑤ ★ AI 에이전트 영속 메모리 (13,673줄)
│   toc.md(목차+소유권) projectbrief techContext systemPatterns decisions(ADR)
│   activeContext progress security(§1~§19) realityCheck backendApi ideas
│   visualStory agentLog
│   tasks/2026-05/ (24개) tasks/2026-06/ (19개)
│   phases/phase12,13,15-plan.md   releases/0.7.0/
│
├── docs/                       PLAN.md PHILOSOPHY.md BUILD.md
│                               release-notes/(12개) screenshots/ icon/
├── landing/                    프로젝트 홍보 정적 사이트
├── scripts/validate-recipes.mjs
├── .github/workflows/          linux-build.yml validate-recipes.yml
├── CLAUDE.md / AGENTS.md / GEMINI.md   ⑥ AI 에이전트 작업 규칙
├── bundles.json                레시피 9개 병합 배포본
├── CONTRIBUTING.md / CONTRIBUTING-bundles.md / SECURITY.md / LICENSE
└── .gitleaks.toml / .gitleaksignore
```

---

## 4. 기능 9개

| 섹션 | 내용 |
|---|---|
| **Dashboard** | 설치 수, 업데이트 대기, brew 버전, formula/cask 분할, 카테고리 도넛차트, 용량(Cellar/Caskroom/var-log/cache) + Finder 열기, `brew doctor` 원클릭, 캐시 정리(회수용량 추정), Exposure 카드(취약점) |
| **Library** | 설치 formula/cask 전체 목록, outdated 배지, 정렬 컬럼, 카테고리 칩 필터, Vulnerable 필터 + 심각도 점, **Pinned 필터 + pin/unpin**(formula와 cask 모두), 슬라이드오버 상세 패널 |
| **Discover** | 전체 카탈로그 16,000+ 검색, 19개 카테고리 타일, 대형 카테고리 서브카테고리 드릴다운, 멀티셀렉트 칩 필터 |
| **Bundles** ⭐ | 사양 게이팅 원클릭 스택 9개. 제로설치 시스템 프로파일(RAM/arch/GPU/디스크)로 **Ready / Marginal / Not recommended** 등급. 의도 문단 + 패키지별 설치 + 셋업 체크리스트(brew-native는 앱 내 실행, 외부 명령은 복사만) |
| **Trending** | `formulae.brew.sh` 공개 분석 데이터, 30/90/365일 윈도우, **velocity index**(최근 1달 vs 이전 11달), 옵트인 스파크라인 |
| **Snapshots** | `brew bundle`로 Brewfile 저장/복원 — "새 맥 원클릭 세팅" |
| **Services** | launchd 서비스 목록/시작/정지/재시작 (`brew services`) |
| **Security** | 옵트인 CVE 스캔 (`brew vulns` → OSV.dev). GitHub 로그인 시 GHSA 상세 enrichment. "Upgrade to fix" 기존 업그레이드 파이프라인 연결. 기본 꺼짐 + `brew vulns` 원클릭 설치 |
| **Activity** | 모든 `brew` 호출 stdout/stderr 실시간 스트리밍, 리사이즈 가능 하단 드로어, 최근 200잡 영속 |

**키보드**: `Cmd+K` 커맨드 팔레트 / `Cmd+0` Dashboard / `Cmd+1~7` 섹션(Bundles=⌘7) / `Cmd+,` Settings

### 번들 9개

| ID | 이름 | 구성 |
|---|---|---|
| `local-llm` | Local LLMs | ollama + open-webui |
| `image-gen` | Image Generation | ComfyUI (SD/Flux) |
| `graphics` | Graphics & Design | Inkscape, GIMP, Krita |
| `media` | Media Toolkit | ffmpeg, yt-dlp, mpv |
| `web-dev` | Web Dev Starter | node + pnpm |
| `databases` | Local Databases | PostgreSQL + Redis |
| `agentic-web-dev` | Agentic Web Dev | opencode, zed, node, pnpm, git, caddy, orbstack |
| `lamp` | LAMP Stack | Apache + MySQL + PHP |
| `lemp` | LEMP Stack | Nginx + MariaDB + PHP-FPM |

`recipes/local-llm.json`의 `capabilityNotes` — RAM별 실전 조언:
`8`→"7~8B Q4만, 더 크면 스왑" / `16`→"13~14B 편안" / `24`→"26~32B급" / `48`→"70B"

---

## 5. 철학 & 보안 포스처

### 선언 (`README.md`, `docs/PHILOSOPHY.md`)

```
MIT licensed. No CLA. No EULA. No telemetry. No account. No dark patterns.
"Not freemium — there is no paid tier, because there is no tier."
"modern developer tools are increasingly asking users to trade transparency
 for convenience. We reject that trade."
```

- Homebrew 대체품 아님 — 터미널이 진실의 원천(source of truth)
- local-first / inspectable / deterministic / 사용자 주체성 존중

### 네트워크 호출 12가지 전부 문서화

**항상 켜짐 (Homebrew 공식 등)**
1. `formulae.brew.sh/api/analytics/install` — Trending (의존성 포함)
2. `formulae.brew.sh/api/analytics/install-on-request` — 직접 설치분
3. `formulae.brew.sh/api/{formula,cask}.json` — 카탈로그 (내장, 수동 새로고침 시)
4. Cask 홈페이지 아이콘 프로브 — 주 1회 제한, 7일 캐시(미스 포함), SSRF 방어
5. `brew` 자체 통신 — 터미널 직접 실행과 동일
6. 기본 브라우저 열기 — http(s) 외 스킴 거부

**기본 꺼짐 (옵트인)**
7. `api.github.com/repos/...` — 패키지 GitHub 통계 (24h 캐시)
8. `github.com/login/{device,oauth}/*` — OAuth Device Flow 로그인
9. `api.github.com/{user/starred, .../subscription, .../issues}` — Star/Watch/Issue
10. `brew-browser.zerologic.com/updater.json` + GitHub 릴리스 아티팩트 — 자체 업데이터
11. `brew-browser.zerologic.com/trending-history/*` — 추이 스파크라인
12. `brew-browser.zerologic.com/enrichment/*` — 카테고리/설명 최신화
13. `api.osv.dev` + 소스포지 + `api.github.com/advisories/{GHSA}` — CVE 스캔

**Offline Mode** 한 번으로 전부 차단. 설정 파일 손상/누락 시 **fail closed**
(오프라인 모드로 동작). 프론트엔드 `fetch()` 0개 — 전부 타입드 Tauri IPC.

### 보안 성적 (`memory-bank/security.md`)

```
verdict: READY-FOR-SCRUTINY (0 critical / 0 high / 0 medium / 0 low / 0 nit)
초기 감사 16개 발견 전부 verified-fixed + 테스트 통과

cargo audit        0 vulns (566 crates)
cargo deny         advisories + bans + licenses + sources ok
npm audit          0 vulns (--omit=dev)   / 개발의존성 3 low (cookie<0.7 via SvelteKit)
semgrep            0 findings (security-audit + OWASP-top-10 + Rust + TS)
cargo clippy -D warnings   clean
unsafe Rust        0개
@html/innerHTML/eval  0개
tauri-plugin-shell    미사용
SSRF 방어          link-local/loopback/RFC1918/클라우드메타데이터 차단
                   + 리다이렉트 매 홉 재검사
파서               양쪽 언어 fuzz 테스트
2026-06-07         양쪽 빌드 사전 릴리스 보안 패스 — clean (security.md §19)
```

### CSP (`src-tauri/tauri.conf.json`)

```
default-src 'self'; connect-src 'self' formulae.brew.sh api.github.com
github.com brew-browser.zerologic.com objects.githubusercontent.com;
img-src 'self' data:; object-src 'none'; base-uri 'self'; frame-ancestors 'none'
```

### 권한 (`src-tauri/capabilities/default.json`) — 9개만

`core:default` `opener:default` `core:event:default`
`core:window:allow-start-dragging` `core:window:allow-set-theme`
`dialog:allow-open` `dialog:allow-save` `updater:default` `window-state:default`

---

## 6. Q&A

### Q. 설치 및 사용법?

**Homebrew 설치 (권장)**
```sh
brew tap msitarzewski/brew-browser
brew trust msitarzewski/brew-browser   # Homebrew 6.x 비공식 tap 신뢰 필요 (1회)
brew install --cask brew-browser
brew upgrade --cask brew-browser       # 나중에 업데이트
```

**직접 다운로드**

| 환경 | 파일 |
|---|---|
| Apple Silicon, macOS 13+ | `brew-browser_<ver>_aarch64.dmg` |
| Intel, macOS 13+ | `brew-browser_<ver>_x64.dmg` |
| macOS 26 네이티브 | `BrewBrowser-<ver>-arm64.dmg` / `-x86_64.dmg` |
| Linux (Ubuntu 22.04+) | `.deb` / `.rpm` / `.AppImage` |

macOS 빌드는 서명 + 노타라이즈 → Gatekeeper 경고 없음.
**Linux 주의**: 아티팩트 현재 미서명(체크섬 검증 필요) / GitHub 로그인은
Secret Service 데몬(gnome-keyring, KWallet) 필요, 없으면 로그인만 실패.

**소스 빌드**
```sh
git clone https://github.com/bmshin94/brew-browser
cd brew-browser && npm install
npm run tauri dev      # HMR 개발
npm run tauri build    # macOS .dmg / Linux .deb .rpm .AppImage

rustup target add x86_64-apple-darwin                   # Intel 크로스빌드
npm run tauri build -- --target x86_64-apple-darwin

cd native && swift build && ./build-app.sh && open BrewBrowser.app   # Swift 빌드
```
사전 준비: Rust stable, Node 22+, Homebrew,
macOS는 `xcode-select --install`, Linux는 `.github/workflows/linux-build.yml`의
`apt-get install` 블록 그대로 사용(README가 2중 목록 유지 금지 명시)

**추천 첫 사용 순서**: Dashboard 현황 → Settings→Network 검토 → Library 정리 →
**Snapshots 백업 먼저!** → Bundles 등급 확인 → Activity 드로어 구경

---

### Q. 플러그인? 스킬? MCP?

**전부 아님. 독립 실행 데스크톱 앱.**

| 종류 | 해당? | 검증 |
|---|---|---|
| Plugin | ❌ | `.claude-plugin/`, `plugin.json` 없음 |
| Skill | ❌ | `SKILL.md` 없음 |
| MCP Server | ❌ | `@modelcontextprotocol` 없음, `mcp.json` 없음 |
| CLI | ❌ | bin 엔트리 없음 |
| **Desktop App** | ✅ | `tauri.conf.json`에 창 크기(1100x720) + macOS Developer ID 서명 |

**혼동 원인**: 루트에 `CLAUDE.md`(37KB) / `AGENTS.md`(35KB) / `GEMINI.md` +
`memory-bank/`(13,673줄)가 있어서. 하지만 이건 **"AI가 이 앱을 개발할 때의 규칙"** —
AI가 소비하는 도구가 아니라 **AI가 생산한 결과물**.

`memory-bank/toc.md` 명시: *"Root `.md` 파일은 GitHub 관례에 AI 워크플로우 파일만 추가"*

**AI 연동을 원한다면** 별도 MCP 서버를 만들어야 하며, 이 레포가 최적 재료:
`src-tauri/src/commands/` 85개가 이미 "brew 안전 API" 완성본.
특히 `brew/exec.rs`(안전 실행) + `parse.rs`(출력 파싱, fuzz 테스트 완료) +
`error_patterns.rs`(에러 분류)가 핵심 자산.

---

### Q. API 토큰 필요?

**아니. 기능의 약 95%가 토큰 0개로 동작.**

**토큰 불필요**: Dashboard, Library, Discover(16k 검색), 설치/삭제/업그레이드,
Snapshots, Services, Bundles, Activity, 카탈로그 새로고침, 자체 업데이트,
**Trending**(공개 JSON — README: *"no API key, no account"*),
**CVE 스캔**(`brew vulns` → OSV.dev 공개 API)

**GitHub 통합만 선택적** — 그리고 **사용자가 토큰을 발급/붙여넣지 않음**:
```
OAuth Device Flow (RFC 8628)
1. 앱이 코드 표시 (예: ABCD-1234)
2. 실제 브라우저로 github.com/login/device
3. 코드 입력 → GitHub에서 직접 승인
4. 앱은 토큰만 수신 — 비밀번호 미노출

임베디드 웹뷰 없음 / client secret 없음 / callback URL 없음
scope: read:user + public_repo + notifications (최소)
저장: macOS 키체인 / Linux Secret Service
  키: com.zerologic.brew-browser/github_access_token
"토큰은 프론트엔드로 반환 안 됨, 디스크에 안 씀, 로그에 안 남음 — 유닛테스트 검증"
```

**로그인 효과**: GitHub API 한도 60/시간 → **5,000/시간**,
Star/Watch/Issue 작성 가능, CVE의 GHSA 상세정보

**보안 장치**: ① scope 사전 검증(GitHub 요청 전 `scope_required` 실패)
② URL 엄격 허용목록(gist/raw/접미사공격/경로순회 거부)
③ GHSA 요청은 Offline OFF + vuln ON + GitHub ON **삼중 조건**

> 참고: `recipes/agentic-web-dev.json`의 `opencode`는 모델 제공자 설정 필요.
> 단 이는 그 도구의 요구사항이며 로컬 Ollama 사용 시 API 키 불필요.

---

### Q. 왜 GitHub에서 유명한가

(정확한 스타 수는 이 세션 권한 범위에서 확인 불가 — 레포 내용 기반 요인 분석)

| # | 요인 | 근거 | 무게 |
|---|---|---|---|
| 1 | 수요 큰데 공급 없던 자리 | Homebrew는 맥 개발자 필수, 쓸만한 GUI가 계속 없었음 | 🔥🔥🔥🔥🔥 |
| 2 | "정직한 오픈소스" 서사 | "No CLA/EULA/telemetry/account/dark patterns", "there is no tier", PHILOSOPHY "We reject that trade" | 🔥🔥🔥🔥🔥 |
| 3 | 극단적 투명성 | 네트워크 호출 12개 URL·시점·데이터·캐시·기본값 전부 명시, 신뢰경계 자진 구분, IP 미로깅을 Caddy 설정 공개로 감사 가능화(security.md §16), "Re-audits are welcome" | 🔥🔥🔥🔥 |
| 4 | 보안 성적 압도적 | 0/0/0/0/0, unsafe 0, shell plugin 미사용, SSRF 홉별 재검사, fuzz | 🔥🔥🔥🔥 |
| 5 | 두 빌드 기술 야심 | Rust 26.7k + Swift 18.4k 병렬 구현 + 데이터 계약 패리티, Liquid Glass 실전 레퍼런스 | 🔥🔥🔥 |
| 6 | Bundles 공감 기능 | "8GB 맥에 못 돌릴 LLM 스택 권하지 않음" + RAM별 실전 조언 | 🔥🔥🔥 |
| 7 | AI 워크플로우 공개 | CLAUDE.md 37KB + memory-bank 13.6k줄 + tasks 43개 실기록 | 🔥🔥🔥 |
| 8 | 기여 친화 | 레시피=JSON+스키마+CI자동검증, No CLA, cask 미지원을 가짜 clean 없이 정직 표시 | 🔥🔥 |

---

### Q. 로컬 에이전트 구축에 도움?

**매우 큰 도움. 단 "에이전트 코드"는 없음(LLM 호출 0개) — "방법론"이 자산.**

#### 보물 1: `CLAUDE.md` = 에이전트 제어 아키텍처 설계도

| 요소 | 내용 |
|---|---|
| **State Machine** | `PLAN → BUILD → DIFF → QA → APPROVAL → APPLY → DOCS`, substates `CODING/WAITING_TOOL/RUNNING/IDLE`, 각 상태 In/Out/Exit/Failure 정의 |
| **승인 게이트** | 승인 없이 파일 변경 금지 / 코드 승인 전 문서 금지 / BUILD는 diff 생성만(**NOT APPLIED**) / 명시적 키워드까지 IDLE |
| **Budget** | cycles 3 / tokens 100k / minutes 30 + 초과 시 대응(STALL, 최소컨텍스트, 연장요청) |
| **Stall Detection** | 연속 2회 동일 diff → 감지·로그·중단·진단보고·방향요청 |
| **Compaction Protocol** | "압축은 예고 없다 → 모든 전환마다 continuous 저장". 5종 저장(상태위치/진행/결정/전환로그/**Loose context**) + `COMPACTION RECOVERY` 복구 |
| **Four Sacred Rules** | 재사용분석 없는 새 파일 금지 / 재작성 금지 / 일반론 금지(`file:line` 필수) / 기존 아키텍처 무시 금지 + Reuse Validation Checklist |
| **절대 금지** | 프로덕션 fake/mock 데이터, 스텁을 완료 처리, 테스트 실패 무시, "방어적 프로그래밍"(근본원인 고쳐라), 무승인 적용 → 전부 롤백+재시작 |
| **병렬 / Agent Swap** | 독립작업 병렬 스폰 + 의존작업 후행 / 깨끗한 경계에서 focused context로 전문 에이전트 이관 |

#### 보물 2: `memory-bank/` = 검증된 영속 메모리 아키텍처

벡터DB/RAG 없이 **구조화 마크다운 + 소유권 + 읽기·쓰기 규칙**으로 해결.
13,673줄이 실제 축적되어 0.7.2 출시까지 운영됨.

```
계층1 거의 불변: projectbrief productContext techContext projectRules
계층2 패턴발견시: systemPatterns decisions(ADR)
계층3 자주변경:  activeContext progress tasks/YYYY-MM/

로딩 모드 3단:
  Fast Track       버그픽스 → 현재월 README + quick-start
  Standard         기능/테스트 → + projectbrief systemPatterns techContext
                                    activeContext progress + docs/ 스캔
  Deep Dive        아키텍처 → + 특정월 README + decisions.md

협업 규칙(toc.md):
  "Read first" — 최소 NEXT-SESSION, projectbrief, activeContext, decisions
  "Write only your owned files" — 남의 명세 수정 금지,
                                  // REQUEST FROM <agent>: 로 요청
  "No code generation without spec" — 명세 없으면 명세부터
컨텍스트 로테이션: 전환 후 Task Context 폐기, Core만 유지
```

#### 보물 3: `src-tauri/` = 안전한 툴 실행 레퍼런스

```
brew/exec.rs         셸 미경유 + 타입드 enum 인수배열 + tokio::process
                     + 타입드 Channel 스트리밍
  ❌ exec(`brew install ${userInput}`)    → "wget; rm -rf ~" 위험
  ✅ Command::new("brew").args(["install", &validated])  → 주입 불가
brew/parse.rs        CLI 출력 파싱 + fuzz + tests/fixtures/ 실출력 10종
brew/error_patterns.rs  CLI 에러 → 타입드 에러 (에이전트 복구 가능성)
system/profile.rs    RAM/arch/GPU/디스크 → "이 기계에서 가능한가" 판단
commands/ 85개       툴 세분화 granularity의 실전 균형점
capabilities/default.json  권한 화이트리스트 9개
```

#### 보물 4: 번들 레시피 = 에이전트 실행 환경

```
local-llm       ollama(localhost:11434, OpenAI 호환) + open-webui
agentic-web-dev opencode(모델 무관: Anthropic/OpenAI/Google/로컬 Ollama)
                + zed + node + pnpm + git + caddy + orbstack
→ 합치면: 완전 로컬 / API 비용 0 / 데이터 외부유출 0 에이전트 환경
capabilityNotes로 RAM별 가능 모델 크기 판단
```

#### 실전 로드맵
```
Phase 0 환경: Bundles로 Local LLMs 등급 확인 → Ollama+Open WebUI
              → Agentic Web Dev로 opencode → 로컬 Ollama 연결
Phase 1 규칙: State Machine + Budget + Stall Detection 이식 (승인게이트 필수 유지)
Phase 2 메모리: memory-bank/ 생성 + 3단 로딩 모드 구현
Phase 3 툴: exec.rs 패턴(셸 미경유/타입드/화이트리스트) + parse.rs + error_patterns.rs
Phase 4 선택: 85개 커맨드를 MCP tool로 노출 → Homebrew MCP Server
```

| 관점 | 도움 정도 |
|---|---|
| 에이전트 제어 아키텍처 | ⭐⭐⭐⭐⭐ |
| 영속 메모리 설계 | ⭐⭐⭐⭐⭐ |
| 안전한 툴 실행 패턴 | ⭐⭐⭐⭐⭐ |
| 에이전트 실행환경 세팅 | ⭐⭐⭐⭐ |
| MCP 서버 재료 | ⭐⭐⭐⭐ |
| 바로 쓸 에이전트 코드 | ⭐ (없음) |

---

### Q. React나 PHP로 만들 수 있나?

| 만들 것 | 기술 | 가능성 | 재사용률 | 추천 |
|---|---|---|---|---|
| React로 프론트 교체 | React + Zustand + Tauri | ✅ 쉬움 | 60%+ | ⭐⭐⭐⭐ 학습용 최고 |
| PHP 웹으로 brew 관리 | PHP + SSE | 🟡 제약 큼 | 30% | ⭐⭐ 학습용만 |
| PHP 데스크톱 앱 | NativePHP | 🟡 미성숙 | 20% | ⭐ |
| **PHP 팀 관리 서버** | **Laravel** | **✅ 최적** | **보완 관계** | **⭐⭐⭐⭐⭐** |
| 다른 생태계 GUI | Tauri 재사용 | ✅ 좋음 | 70% | ⭐⭐⭐⭐⭐ |

#### React 프론트 교체 — 가능. Tauri는 프레임워크 무관

```
변경 없음:  src-tauri/ (26,782줄 Rust 전부)
거의 복사:  src/lib/api.ts, types.ts  (@tauri-apps/api는 순수 JS, 프레임워크 무관)
그대로:     src/lib/styles/*.css (tokens/typography/reset)
변환 필요:  컴포넌트 44개(17,698줄), 스토어 23개
설정 3줄:   tauri.conf.json의 beforeDevCommand / devUrl / frontendDist

runes → React 매핑
  $state    → useState        $derived → useMemo
  $effect   → useEffect       stores   → Zustand (사고방식 가장 유사)
  {#if}     → {cond && }      {#each}  → .map()
  onclick   → onClick         bind:value → value + onChange
  <slot/>   → {children}      $props() → props
의존성: @lucide/svelte → lucide-react (같은 아이콘셋)
```

#### PHP 웹의 근본 제약 4가지
```
1. 실행 위치 — 내 맥의 brew를 만지려면 내 맥에서 PHP가 돌아야 함
               → localhost 접속 형태, 앱 더블클릭 경험 아님
2. 권한 — www-data/_www 유저 ≠ 사용자 홈/권한
          → php -S 로 내 유저 권한 직접 기동
3. 스트리밍 — brew install 장시간 출력 → SSE/WebSocket 필요
               (Tauri는 typed Channel로 기본 제공)
4. 보안 ★ — 웹 인터페이스로 시스템 명령 실행은 HTTP 공격표면 생성
             CSRF로 악성 사이트가 로컬 서버 호출 가능
필수 완화: 127.0.0.1 바인딩만 / CSRF 토큰 / Origin 검증 /
          패키지명 정규식 화이트리스트 / proc_open 배열형만
          (shell_exec·exec·system 금지) / 로컬 토큰 인증
```

#### PHP의 진짜 자리 = 팀/기업 서버 (Laravel)
```
[각 맥] brew-browser 수정판 ──HTTPS+디바이스토큰──▶ [Laravel]
                                                    ├ 웹 대시보드
                                                    ├ MySQL: devices snapshots
                                                    │   baselines allowlists
                                                    │   vulns audit_logs licenses
                                                    └ Queue/Scheduler: CVE·알림
클라이언트 추가 커맨드 (src-tauri/src/commands/team.rs 신규):
  push_snapshot_to_team / pull_team_baseline
  check_package_allowed / report_vulns_to_team
Laravel 이점: Eloquent 스키마 / Sanctum 디바이스토큰 / Horizon 큐 /
              Blade·Livewire 초고속 대시보드 / 배포 저렴
```

---

## 7. 수익화 전략

### 전제 3가지

```
1. MIT 라이선스 — 상업이용/수정/배포/비공개 사용 전부 허용.
   의무는 저작권 표시 + 라이선스 사본뿐. No CLA. GPL 아님 → 비공개 유료 가능
2. 원본에 유료 티어를 붙이면 안 됨 — "there is no tier" 서사가 프로젝트 자산
3. 올바른 전략 = 무료 앱은 유입 채널, 유료는 "옆"에 별개 레이어
   (Open Core / Commercial Companion — Sentry, GitLab, Grafana 방식)
```

### 아이디어 7개

| # | 아이디어 | 난이도 | 수익성 | 적합도 |
|---|---|---|---|---|
| 1 | **BrewFleet — 팀/기업 환경관리 SaaS** | 🔨🔨🔨🔨 | 💰💰💰💰💰 | ⭐⭐⭐⭐⭐ |
| 2 | **Agent Workflow Kit — 템플릿/강의/컨설팅** | 🔨 | 💰💰💰 | ⭐⭐⭐⭐⭐ |
| 3 | 다른 생태계 확장 (npm/pip/apt/winget) | 🔨🔨🔨 | 💰💰💰 | ⭐⭐⭐⭐ |
| 4 | Recipe Hub — 번들 마켓플레이스 | 🔨🔨 | 💰💰💰 | ⭐⭐⭐⭐ |
| 5 | Homebrew MCP 서버 | 🔨🔨 | 💰💰 | ⭐⭐⭐⭐ |
| 6 | CVE/컴플라이언스 모니터링 | 🔨🔨🔨 | 💰💰💰💰 | ⭐⭐⭐ |
| 7 | 후원 / 부트스트랩 | 🔨 | 💰 | ⭐⭐ |

---

#### 1. BrewFleet — 팀/기업용 개발환경 관리 SaaS

**해결 문제**: 신입 온보딩 삽질("제 맥에서만 안 돼요") / 보안팀이 조직 전체
취약 패키지 현황 파악 불가 / SOC2·ISO27001 감사용 소프트웨어 인벤토리 수동 수집

**티어**
| 티어 | 가격 | 핵심 |
|---|---|---|
| Free | $0 (3명) | 스냅샷 백업, 팀원 환경 목록 — 바이럴 유입 |
| Team | $6/user/월 | Brewfile 중앙저장소+버전관리(diff), 팀 베이스라인, 드리프트 감지, 온보딩 원클릭, Slack 알림, 조직 CVE 집계 |
| Business | $12/user/월 | 승인 패키지 화이트리스트, SSO(SAML/OIDC), **감사 로그(SOC2 필수)**, 라이선스 추적(GPL 감지), 컴플라이언스 PDF, RBAC, CVE SLA 추적 |
| Enterprise | $2,000+/월 | **셀프호스팅**, 프라이빗 tap, 에어갭/미러, 전용지원 SLA, MDM·CMDB 연동 |

**수익 시뮬레이션**
```
1년차 보수적: Team 30팀×8명×$6 + Business 5팀×40명×$12
             = MRR $3,840 → ARR 약 $46,000
2~3년차 중간: Team 150팀 + Business 30팀 + Enterprise 3곳
             = MRR $32,700 → ARR 약 $392,000
```
**리스크**: 기업 영업 3~6개월 / Jamf·Kandji 기능 확장 / 신뢰(회사 머신 정보 전송)
→ 대응: 셀프호스팅 옵션 + 오픈소스 클라이언트 + 원본의 투명성 DNA 계승

---

#### 2. Agent Workflow Kit — **최우선 추천** ⚡

**시장 근거**: 2026년 최대 관심사가 "AI로 프로덕션 앱 가능한가". 대부분 "AI 코딩 규칙"은
10줄 잔소리, 강의는 투두리스트 수준. 이 레포는 **37KB 규칙 + 13.6k줄 메모리 +
43개 실작업 기록 → 6만줄 앱 0.7.2 출시**라는 검증된 사례.

**상품 라인업**
```
A. 템플릿 팩 $49~$99
   agent-workflow-kit/
     AGENTS.md (범용화)
     memory-bank/{_templates, toc.md(소유권 매트릭스), examples}
     adapters/{claude-code, cursor, copilot, cline, aider, opencode}
     scripts/{init-memory-bank.sh, state-transition.sh,
              budget-check.mjs, stall-detect.mjs}
     stacks/{laravel-php, react-typescript, tauri-rust, python-fastapi}
     GUIDE.md (50p)
   채널: Gumroad / Lemon Squeezy / 자체

B. 온라인 강의 $199~$499 (12모듈)
   1 왜 AI가 실패하는가  2 상태머신  3 승인게이트  4 메모리뱅크
   5 예산·막힘감지  6 압축 생존(Loose context)  7 멀티에이전트 조율
   8 안전한 툴 실행(셸 미경유)  9 로컬 에이전트(Ollama+opencode)
   10 케이스스터디(6만줄 추적)  11 PHP/Laravel 적용  12 React/TS 적용
   채널: Udemy(볼륨) / Teachable·Gumroad(마진) / 인프런(한국 선점)

C. 컨설팅 $2,000~$10,000/건 (4주: 진단→설계→워크숍→팔로업)
D. 구독 뉴스레터·커뮤니티 $10~$20/월 (100명 = $1,000~2,000/월)
```
**1년차 예상**: 템플릿 100×$69 + 강의 200×$299 + 컨설팅 5×$5,000 +
구독 150×$15×12 ≈ **$118,000**

**왜 1순위**: 개발 거의 없음 / 2~4주 출시 / 마진 90%+ / 정리하며 본인도 습득 /
한국어 자료 공백 선점 / 나머지 6개의 마케팅 기반 / 실패 손실 최소

**윤리 필수**: 원본 코드 재판매 금지(판매 대상은 방법론·범용 템플릿·강의).
크레딧 명시 — *"derived from the AGENTS.md/memory-bank pattern pioneered in
msitarzewski/brew-browser (MIT)"*. 출처 표기는 오히려 "실전 검증" 세일즈 포인트.

---

#### 3. 다른 생태계 확장 — 재사용률 70%

아키텍처가 Homebrew 전용이 아님. "CLI를 안전하게 감싸고 GUI 붙이는 범용 패턴".
`exec.rs`는 그대로, `parse.rs`/`error_patterns.rs`만 교체, 컴포넌트 44개 거의 그대로.

| 대상 | 규모 | 경쟁 GUI | 기회 |
|---|---|---|---|
| npm/pnpm/yarn | 초거대 | 약함 | ⭐⭐⭐⭐⭐ |
| **pip/uv/conda** | 거대(AI 붐) | 거의 없음 | ⭐⭐⭐⭐⭐ |
| apt/dnf/pacman | 거대 | 낡음 | ⭐⭐⭐⭐ |
| winget/scoop/choco | 거대 | 약함 | ⭐⭐⭐⭐ |
| cargo | 중간 | 없음 | ⭐⭐⭐ |

**특히 pip/uv Browser**: 의존성 지옥 시각화, venv 전환 GUI,
requirements/pyproject 스냅샷, **Bundles 패턴으로 "PyTorch+CUDA 스택" GPU 등급**
(`system/profile.rs`가 이미 GPU 감지), pip-audit CVE 연동
→ 장기적으로 아이디어 1과 통합해 **"DevEnv Fleet"** (전 패키지매니저 통합 플랫폼)

---

#### 4. Recipe Hub — 번들 마켓플레이스

`recipes/recipe.schema.json` 공개 계약 + `scripts/validate-recipes.mjs` +
CI 검증이 이미 있음 → 레시피 허브로 확장

```
웹(Laravel): 브라우징/검색, 사용자 업로드(스키마검증), 별점·리뷰·설치수,
             "내 사양 가능 레시피만" 필터, 버전관리, 딥링크 brewbrowser://recipe/xxx
수익: 프리미엄 레시피(제작자 70/플랫폼 30) $9~$15
      Verified 배지 $29/년
      기업 프라이빗 허브 $99/월 (아이디어 1과 통합)
      스폰서 레시피 (JetBrains, Docker, MongoDB 등)
리스크: 치킨-에그 → 초기 20~30개 직접 제작 필요
```

---

#### 5. Homebrew MCP 서버

85개 커맨드를 MCP tool로 노출. Free=읽기전용 / Pro $5월=쓰기+동기화.

```
🚨 필수 안전장치 (AI가 패키지 설치·삭제는 극도로 위험)
  모든 쓰기에 사용자 승인 (CLAUDE.md APPROVAL 게이트 그대로)
  dry-run 선행 / 삭제 특히 엄격 / Brewfile 자동 백업 후 실행
  ★ 프롬프트 인젝션 방어 — 패키지 설명·README에 악성 지시 가능
  화이트리스트 검증 (exec.rs 패턴)
```
수익성보다 **권위/유입 채널** 가치가 큼 → 무료 공개 후 1·2번으로 연결

---

#### 6. CVE/컴플라이언스 모니터링

현재 스캔은 로컬+수동. 기업은 지속 모니터링+리포트를 원함.
```
에이전트: 주기적 brew vulns → 서버 보고
서버: 조직 CVE 집계, 심각도 트렌드, SLA 추적(critical 7d/high 30d 준수율),
      Slack·PagerDuty 알림, SOC2·ISO27001 PDF 자동생성, GPL 라이선스 감지
가격: Startup $99/월(20머신) / Growth $299/월(100) / Enterprise $999+(무제한+셀프호스팅)
차별화: Snyk·Dependabot·Wiz는 CI·레포 중심 → "개발자 로컬 머신 전문"
정직성: brew vulns는 formula만 지원(cask 미지원) 명시 — 원본 DNA 계승
```

---

#### 7. 후원 / 부트스트랩

원본 방식(`.github/FUNDING.yml`: `github: [msitarzewski]`).
GitHub Sponsors / Open Collective / Ko-fi / 기업 스폰서 티어 $100~$1,000월.
단독 생계 불가하나 다른 아이디어의 신뢰 기반.

---

### 최종 실행 로드맵

```
Phase 1 (1~2개월) 씨앗심기 — 아이디어 2
  W1-2 AGENTS.md 범용화 + 템플릿 정리
  W3-4 무료 버전 GitHub 공개 (유입 채널)
  W5-6 유료 팩 $69 Gumroad 출시
  W7-8 블로그·유튜브·트위터 + 한국 커뮤니티(OKKY, 인프런)
  예상 $2,000~$8,000 / 부수효과: 방법론 습득 + 권위

Phase 2 (2~4개월) 강의+검증 — 아이디어 2 확장 + 5 무료공개
  12모듈 강의 (인프런 + Udemy) / MCP 서버 무료 공개로 스타 확보
  뉴스레터·Discord 시작
  예상 $15,000~$50,000 / 부수효과: 잠재고객 리스트 (Phase 3 연료)

Phase 3 (4~10개월) 본게임 — 아이디어 1 BrewFleet
  M1-2 Laravel API + 대시보드 MVP
  M3   React 클라이언트 (Rust 백엔드 재사용)
  M4   베타 (Phase 1-2 커뮤니티에서 모집)
  M5-6 유료 전환 + 피드백
  예상 MRR $2,000 → $10,000

Phase 4 (10개월+) 확장 — 아이디어 3 + 4 + 6
  "DevEnv Fleet": 전 패키지매니저 통합 플랫폼
  예상 MRR $30,000+

순서 근거: ① Phase1 개발 거의 없어 빠른 검증 ② Phase2가 Phase3 영업비용 0으로
③ Phase3은 쌓인 권위로 신뢰도↑ ④ 각 단계가 다음 단계 마케팅 채널(복리)
⑤ Phase1-2 실패 시 손실 최소
```

### 스킬 매칭

| 아이디어 | React | PHP/Laravel | Rust(학습) | 즉시성 |
|---|---|---|---|---|
| 1 BrewFleet | ✅✅ | ✅✅✅ | 🟡 | 🔨🔨🔨🔨 |
| 2 Workflow Kit | — | — | — | **✅ 지금** |
| 3 생태계 확장 | ✅✅ | 🟡 | ✅✅ | 🔨🔨🔨 |
| 4 Recipe Hub | ✅ | ✅✅✅ | — | 🔨🔨 |
| 5 MCP 서버 | — | 🟡 | ✅✅ | 🔨🔨 |
| 6 CVE 모니터링 | ✅ | ✅✅✅ | 🟡 | 🔨🔨🔨 |
| 7 후원 | — | — | — | ✅ 지금 |

### 윤리 체크리스트

- [x] MIT 라이선스 고지 + 원저작자 크레딧 유지
- [x] 원본의 "정직함" DNA 계승 (텔레메트리 명시, 네트워크 문서화, 다크패턴 없음)
- [x] 별도 브랜딩 ("brew-browser Pro" ❌ → "BrewFleet" ✅)
- [x] 유용한 개선은 upstream 기여 환원
- [x] 강의·템플릿에 출처 명확 표기
- [x] 원본 프로젝트 스폰서

---

## 8. 핵심 요약

```
정체       독립 실행 데스크톱 앱 (플러그인/스킬/MCP 아님)
규모       약 84,000줄, 두 빌드(Tauri+Swift) 병렬 구현
라이선스   MIT / No CLA / No EULA / No telemetry / No account / No tier
토큰       핵심 기능 전부 불필요. GitHub만 선택(Device Flow, 비번 미노출)
보안       0/0/0/0/0, unsafe 0, shell plugin 미사용, SSRF 홉별 재검사
유명 이유  ① 수요-공급 공백 ② 정직함 서사 ③ 네트워크 12개 전부 공개
           ④ 보안 성적 ⑤ 두 빌드 야심 ⑥ Bundles 사양 등급 ⑦ AI 워크플로우 공개
최대 가치  CLAUDE.md(에이전트 제어) + memory-bank(영속 메모리)
           + brew/exec.rs·parse.rs·error_patterns.rs(안전한 툴 실행)
React      ✅ 60%+ 재사용, Rust 백엔드 무수정
PHP        웹 GUI는 제약 많음 → Laravel 팀 관리 서버가 최적 자리
수익화     1순위 Agent Workflow Kit (2~4주, $118k/년 가능)
           장기 BrewFleet SaaS (ARR $392k 가능)
```

### 참고 파일 (읽을 순서)

1. `README.md` — 기능 + 네트워크 포스처 + 보안
2. `docs/PHILOSOPHY.md` — 왜 이렇게 만들었나
3. `CLAUDE.md` / `AGENTS.md` — ★ 에이전트 워크플로우 (최대 자산)
4. `memory-bank/toc.md` — ★ 메모리뱅크 구조 + 소유권 + 협업 규칙
5. `memory-bank/decisions.md` — ADR (두 빌드 유지 근거 등)
6. `memory-bank/security.md` — §1~§19 보안 감사
7. `src-tauri/src/brew/{exec,parse,error_patterns}.rs` — ★ 안전한 CLI 실행
8. `src-tauri/src/lib.rs:166` — 85개 커맨드 등록
9. `src/lib/api.ts` — 타입 안전 IPC 래퍼
10. `recipes/recipe.schema.json` + `recipes/*.json` — 번들 계약
11. `docs/PLAN.md` — 전체 설계 + 페이즈 트래커
12. `memory-bank/backendApi.md` — IPC 전체 표면

---

*이 문서는 세션 분석 기록이며 앱 코드에 영향을 주지 않습니다.*
*원본 프로젝트: https://github.com/msitarzewski/brew-browser (MIT)*
