# MCT Plugin 설정 가이드

이 문서는 `mct-plugin`을 처음 설정하는 사용자를 위한 상세 가이드입니다.

## 📋 요구사항 확인

설정 전에 다음을 준비하세요:

- ✅ **Claude Code** 설치 (CLI, 웹, 또는 IDE 확장)
- ✅ **Linear 계정** (팀 멤버여야 함)
- ✅ **Git 설치** (`git --version` 확인)
- ✅ **gh CLI 설치** (PR 생성용, `gh version` 확인)

### gh CLI 설치 안내

**macOS:**
```bash
brew install gh
```

**Linux:**
```bash
sudo apt-get install gh  # Debian/Ubuntu
sudo dnf install gh      # Fedora/RHEL
```

**Windows:**
```bash
choco install gh         # Chocolatey 사용 시
```

[공식 설치 가이드](https://cli.github.com)

---

## 🔑 Step 1: Linear API 키 발급

### 1.1 Linear 웹 접속
1. [https://linear.app](https://linear.app) 접속
2. 오른쪽 상단 프로필 메뉴 → **Settings**
3. 왼쪽 사이드바 → **API**

### 1.2 Personal API Key 생성
1. **Personal API Keys** 섹션
2. **Create new key** 버튼 클릭
3. 키 이름 입력 (예: "Claude Code")
4. 생성된 키 **전체 복사** (절대 공유 금지 ⚠️)

### 1.3 키 저장
지금부터 사용할 때까지 안전한 곳에 임시 저장:
```
your-api-key-here
```

---

## ⚙️ Step 2: Claude Code에서 `/ls:setup` 실행

Claude Code에서:
```
/ls:setup
```

### 2.1 API 키 입력
```
Linear API 키를 입력하세요 (Linear 웹 → Settings → API → Personal API Keys):
> [붙여넣기]
```

✅ 성공 시: "API 키 확인: jsdq8sd1sd8..."

❌ 실패 시: "API 키가 유효하지 않습니다"
- 키를 다시 확인하고 다시 시도
- 붙여넣기(Ctrl+V/Cmd+V) 확인

### 2.2 팀 선택
```
팀 목록:
1) Adevi (ADE)
2) Backend (BKD)
3) Frontend (FE)

팀을 선택하세요 (번호):
> 1
```

**선택 기준:**
- 작업할 팀 선택
- 여러 팀에 속하면 주로 사용할 팀 선택

### 2.3 워크플로우 상태 매핑

Linear 팀의 상태 목록이 표시됩니다:

```
상태 목록:
1) Backlog (backlog)
2) Todo (unstarted)
3) In Progress (started)
4) In Review (started)
5) Done (completed)
6) Canceled (cancelled)
7) Duplicate (cancelled)
```

4개 역할에 번호 입력:

```
"In Progress" 상태 번호: > 3
"In Review" 상태 번호: > 4
"Done" 상태 번호: > 5
"Canceled" 상태 번호: > 6
```

**매핑 규칙:**
- In Progress: 작업 시작 상태
- In Review: PR 생성 시 변경할 상태
- Done: 완료 상태
- Canceled: 취소/중단 상태

### 2.4 기준 브랜치 확인

```
감지된 기준 브랜치: main
```

자동 감지되지 않으면 수동 입력:
```
기준 브랜치를 입력하세요 (예: main, master):
> main
```

### 2.5 설정 완료

```
설정 완료!

팀: ADE
기준 브랜치: main

사용 가능한 커맨드:
  /ls:list    — 이슈 목록
  /ls:start   — 작업 시작
```

---

## 🔍 Step 3: 설정 확인

### 3.1 생성된 파일 확인

설정 완료 후 2개 파일 자동 생성:

**프로젝트 설정:**
```
.claude/linear.json
```

**전역 설정 (첫 설정 시만):**
```
~/.config/linear/config.json
```

### 3.2 파일 내용 확인

`.claude/linear.json`:
```json
{
  "team_id": "team-uuid...",
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

✅ 4개 상태가 모두 `id`와 `name` 값을 가지면 설정 완료

---

## 🧪 Step 4: 첫 사용 테스트

### 4.1 이슈 목록 확인
```
/ls:list
```

**예상 출력:**
```
★ ADE-24  In Progress  ade-24-log-level-separation       (나)
  ADE-25  Todo         ade-25-workflow-history-ui         (나)
  ADE-26  Backlog      ade-26-payment-refactor            (kim)
```

❌ "API 키 없음" 에러 → Step 2 다시 확인

### 4.2 작업 시작하기
```
/ls:start ADE-25
```

**예상 출력:**
```
## 현재 작업 세션
이슈: ADE-25 — 워크플로우 히스토리 UI
설명: ...
브랜치: ade-25-workflow-history-ui
```

### 4.3 현재 상태 확인
```
/ls:status
```

---

## 🐛 문제 해결

### "API 키가 유효하지 않습니다"
**원인:** 잘못된 API 키 또는 만료된 키

**해결:**
1. Linear 웹에서 새 API 키 생성
2. 기존 키는 삭제
3. `/ls:setup` 다시 실행

### "프로젝트 설정 없음. /ls:setup을 먼저 실행하세요"
**원인:** `.claude/linear.json` 파일 없음

**해결:**
1. 현재 디렉토리가 프로젝트 루트인지 확인 (`.claude-plugin/` 폴더 있는지 확인)
2. `/ls:setup` 재실행

### "gh 인증이 필요합니다: gh auth login"
**원인:** GitHub CLI 로그인 필요 (PR 생성 시)

**해결:**
```bash
gh auth login
```

브라우저에서 인증 후 완료

### gh CLI가 설치되지 않았습니다
**원인:** GitHub CLI 미설치

**해결:**
위의 "gh CLI 설치 안내" 참고

---

## 📖 다음 단계

✅ 설정 완료 후:

1. **`/ls:list`** — 이슈 목록 확인
2. **`/ls:start ADE-XX`** — 이슈 작업 시작
3. **`/ls:pr`** — PR 생성
4. **`/ls:done`** — 이슈 완료

각 커맨드의 상세 설명은 [README.md](./README.md)를 참고하세요.

---

## 🔐 보안 주의사항

⚠️ **중요:**
- API 키는 절대 공유하지 마세요
- `.claude/linear.json`은 `.gitignore`에 등록되어 있습니다
- `~/.config/linear/config.json`은 로컬 머신에만 저장됩니다

---

## 💬 추가 지원

- Linear 문서: https://linear.app/docs
- Claude Code 문서: https://github.com/anthropics/claude-code
- 이슈 보고: GitHub Issues

---

**설정이 완료되었습니다! 🎉**

지금 바로 `/ls:list`를 실행해보세요.
