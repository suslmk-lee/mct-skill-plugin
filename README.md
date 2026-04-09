# MCT Plugin — Linear Skill Marketplace

Linear 이슈 관리와 Git 브랜치/PR 워크플로우를 통합하는 Claude Code 플러그인입니다.

이슈를 시작하면 Claude가 이슈 설명, 수락 기준을 컨텍스트에 로드합니다. 이후 코드 작성 중 참고할 수 있습니다.

## 🔧 설치

```bash
/plugin marketplace add suslmk-lee/mct-plugin
/plugin install ls@mct-plugin
/reload-plugins
```

## 🚀 초기 설정

```bash
/ls:setup
```

Linear API 키, 팀, 워크플로우 상태 매핑, Slack Webhook을 설정합니다.

## 🎯 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/ls:setup` | 초기 설정 |
| `/ls:start [ISSUE-KEY]` | 이슈 시작 — 이슈 컨텍스트 로드 + 브랜치 생성 |
| `/ls:list` | 이슈 목록 (내 이슈 우선) |
| `/ls:done [ISSUE-KEY]` | 이슈 완료 처리 |
| `/ls:pr` | PR 생성 |
| `/ls:scan` | 프로젝트 분석 후 이슈 일괄 등록 (팀장만) |
| `/ls:status` | 현재 브랜치 상태 |
| `/ls:sub [PARENT-KEY]` | 서브이슈 등록 |
| `/ls:integrate [PARENT-KEY]` | 서브이슈 통합 및 PR 생성 |

### `/ls:scan` — 레포지토리 분석 후 선택적 이슈 등록

팀장만 사용할 수 있는 고급 기능으로, 현재 레포지토리를 자동으로 분석하여 기능 개선, 코드 품질, 문서화 등의 항목을 발견하고 **선택적으로** Linear 이슈로 등록합니다.

#### 분석 방식

- **Phase 1 (규칙 기반)**: 파일 구조, 문서, 테스트, 코드 크기 등을 정적으로 분석 (빠름)
- **Phase 2 (LLM)**: Phase 1에서 발견한 항목만 Claude로 깊이 있게 분석 (깊음)

#### 기능

- 📊 6가지 카테고리 자동 검사 (문서화, 테스트, 코드품질, 의존성, 구조, 자동화)
- 🔐 팀장 전용 (권한 검증)
- ✅ 부모-자식 계층 구조로 이슈 생성
- 📋 프리뷰 후 선택적 생성 (승인/거부/연기)
- 🔄 재스캔 지원 (변경사항 자동 감지)
- 💾 결과 저장 및 히스토리 추적

#### 사용 예시

```bash
# 레포지토리 분석
/ls:scan

# 결과 미리보기
/ls:scan --report

# 이전 스캔과 비교
/ls:scan --compare 2026-04-02

# 특정 카테고리만 재분석
/ls:scan --category=tests
```

#### 권한 설정

초기 설정 시:
```bash
/ls:setup
# Step 5에서 "스캔 권한 활성화?" 질문에 [Y] 선택
```

## 📋 시작하기

### 1단계: 저장소 클론 또는 설치
```bash
# Claude Code에서 플러그인 설치
git clone https://github.com/suslmk-lee/mct-plugin.git
```

### 2단계: Linear API 키 준비
- [Linear 웹](https://linear.app) → Settings → API → Personal API Keys
- 새 API 키 생성 후 메모

### 3단계: 초기 설정 실행
Claude Code에서:
```
/ls:setup
```

설정 과정:
1. Linear API 키 입력
2. 팀 선택 (예: Adevi, Backend 등)
3. 워크플로우 상태 매핑 (In Progress, In Review, Done, Canceled)
4. Git 기준 브랜치 선택 (main, master 등)

## 📁 프로젝트 구조

```
mct-plugin/
├── README.md                        # 이 파일
├── SETUP.md                         # 상세 설정 가이드
├── .gitignore                       # Git 제외 파일
├── docs/                            # 설계 & 계획 문서 (git 추적)
│   └── superpowers/
│       ├── specs/
│       │   └── 2026-04-09-ls-scan-expansion-design.md
│       └── plans/
│           └── 2026-04-09-ls-scan-implementation.md
├── tests/                           # 테스트 케이스
│   └── ls_scan_test.md
├── .claude/
│   ├── linear.json.template         # 설정 템플릿 (미포함)
│   ├── linear.json.example          # 설정 예시 (미포함)
│   ├── linear.json                  # 프로젝트 설정 (git 제외)
│   └── scan-results.json            # 스캔 결과 (git 제외)
├── .claude-plugin/
│   └── marketplace.json             # 마켓플레이스 인덱스
└── plugins/ls/
    ├── .claude-plugin/plugin.json   # 플러그인 메타
    ├── commands/                    # 커맨드 정의 (9개 .md)
    │   ├── setup.md                 # /ls:setup
    │   ├── start.md                 # /ls:start
    │   ├── list.md                  # /ls:list
    │   ├── done.md                  # /ls:done
    │   ├── pr.md                    # /ls:pr
    │   ├── scan.md                  # /ls:scan ⭐ NEW
    │   ├── status.md                # /ls:status
    │   ├── sub.md                   # /ls:sub
    │   └── integrate.md             # /ls:integrate
    └── skills/ls/
        ├── SKILL.md                 # 스킬 정의 및 규칙
        ├── auth.md                  # 권한 검증 ⭐ NEW
        └── analyzers/               # 분석 모듈들 ⭐ NEW
            ├── phase1-rules.md      # Phase 1: 규칙 기반 분석
            ├── phase2-llm.md        # Phase 2: LLM 상세 분석
            └── comparator.md        # 재스캔 & 상태 추적
```

## 🔄 워크플로우 예시

### 이슈 시작부터 PR까지

```bash
# 1. 할당된 이슈 목록 확인
/ls:list

# 2. 특정 이슈로 작업 시작 (브랜치 자동 생성, 상태 변경)
/ls:start ADE-24

# 3. 코드 작성... (세션 컨텍스트에서 수락기준 확인)

# 4. PR 생성 (수락기준 체크리스트 자동 포함)
/ls:pr

# 5. PR 머지 후 이슈 완료 처리
/ls:done ADE-24
```

### 서브이슈 워크플로우

```bash
# 1. 부모 이슈에 서브이슈 등록
/ls:sub ADE-20

# 2. 각 서브이슈별로 작업
/ls:start ADE-25
/ls:start ADE-26
# ... 코드 작성

# 3. 모든 서브이슈 완료 후 통합
/ls:integrate ADE-20  # 부모 PR 자동 생성
```

## ⚙️ 설정 파일

### `.claude/linear.json`
자동 생성되는 프로젝트 설정 파일:

```json
{
  "team_id": "LINEAR_TEAM_UUID",
  "team_key": "ADE",
  "base_branch": "main",
  "state_mapping": {
    "in_progress": { "id": "...", "name": "In Progress" },
    "in_review": { "id": "...", "name": "In Review" },
    "done": { "id": "...", "name": "Done" },
    "canceled": { "id": "...", "name": "Canceled" }
  }
}
```

**위치:**
- 프로젝트별: `.claude/linear.json` (`.gitignore`에 등록됨)
- 전역: `~/.config/linear/config.json` (API 키만)

## 🛠️ 요구사항

- **Claude Code** (CLI 또는 웹)
- **Linear 계정** + API 키
- **Git** 설치
- **gh CLI** (PR 생성 시 필요)

## 📚 추가 문서

- **[SETUP.md](./SETUP.md)** — 상세 설정 가이드 및 문제 해결
- **`plugins/ls/commands/`** — 각 커맨드별 실행 로직 상세 설명
- **`plugins/ls/skills/ls/SKILL.md`** — 스킬 규칙 및 API 호출 가이드

## 🚀 다음 스킬 추가하기

`mct-plugin`은 여러 스킬을 지원하도록 설계되었습니다.

### 새 스킬 추가 방법

```
plugins/[new-skill]/
├── .claude-plugin/plugin.json
├── commands/
│   ├── cmd1.md
│   └── cmd2.md
└── skills/[new-skill]/
    └── SKILL.md
```

1. `plugins/` 폴더 아래 새 폴더 생성
2. `.claude-plugin/plugin.json` 작성
3. 커맨드별 `.md` 파일 작성
4. `marketplace.json` 업데이트

## 📝 라이선스

MIT License

## 🤝 기여

이슈 및 PR 환영합니다!

---

**시작하기:** [SETUP.md](./SETUP.md)를 참고하여 초기 설정을 완료하세요.
