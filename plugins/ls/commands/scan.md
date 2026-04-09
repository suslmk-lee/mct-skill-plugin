# /ls:scan — 레포지토리 분석 후 선택적 이슈 생성

You are executing `/ls:scan`. Follow these steps in order. This analyzes the repository for improvement areas and creates Linear issues selectively. **Admin-only.**

## Step 1: Check Permissions

Load `.claude/linear.json`. Verify:
- `enable_scan: true` — if false or missing, tell the user: "❌ 스캔 권한이 비활성화되어 있습니다." and stop.

Load the Linear API key (from env or config file). If missing, tell the user: "❌ API 키 없음. /ls:setup을 먼저 실행하세요." and stop.

Call the Linear API to check the current user's role:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ viewer { name teamMembership(teamId: \"TEAM_ID\") { role } } }"}'
```

Extract `viewer.name` and `viewer.teamMembership.role`. If the role is not `admin`, tell the user: "❌ 스캔 기능은 팀장(Admin)만 사용 가능합니다. 현재 역할: {ROLE}" and stop.

Tell the user: "✅ 권한 확인 완료: {USER_NAME} (팀장)"

## Step 2: Check for Previous Scan Results (Optional)

If `.claude/scan-results.json` exists, load it and extract previous findings. Compare with the current scan (after running Phase 1 & 2) to show:
- **NEW** — findings not in the previous scan
- **STILL PENDING** — findings that are still unresolved
- **COMPLETED** — findings that have been resolved (can be marked as done in Linear)

Tell the user: "이전 스캔 결과와 비교 중..."

## Step 3: Phase 1 — Rule-Based Analysis

Perform automated checks on the repository. For each check, store a finding with:
- `id`: unique ID
- `category`: category name (e.g., "Documentation", "Testing", "Configuration")
- `title`: short title
- `severity`: P0, P1, P2, P3
- `flag`: 🔴 (critical) or 🟡 (warning)
- `confidence`: high/medium/low
- `description`: detailed finding

**Checks to run:**

1. **Missing README.md** — `P1 🔴`
   - Run: `test -f README.md || test -f readme.md` (case-insensitive)
   - Title: "Documentation — README 파일 없음"

2. **Missing Test Directory** — `P1 🔴`
   - Run: `test -d test -o -d tests -o -d __tests__`
   - Title: "Testing — 테스트 디렉토리 없음"

3. **TODO/FIXME Comments** — `P2 🟡`
   - Run: `grep -r "TODO\|FIXME" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.py" src/`
   - Extract up to 5 files
   - Title: "Code Quality — TODO/FIXME 주석 발견"

4. **Missing .gitignore** — `P2 🟡`
   - Run: `test -f .gitignore`
   - Title: "Configuration — .gitignore 파일 없음"

## Step 4: Phase 2 — LLM Analysis (Claude Review)

For each flagged finding (🔴 or 🟡) from Phase 1, ask Claude to provide a deeper analysis:
- "이 발견이 정말 문제인가?"
- "해결 방법은?"
- "우선순위 조정 필요?"

Enhance each finding with Claude's assessment and suggested solutions.

## Step 5: Preview & Selection

Display all findings in a hierarchical list (grouped by category):

```
📊 Repository Analysis Report

【Documentation】
 1) 🔴 P1  README 파일 없음
 2) 🟡 P2  TODO/FIXME 주석 발견 (5개 파일)

【Testing】
 3) 🔴 P1  테스트 디렉토리 없음

【Configuration】
 4) 🟡 P2  .gitignore 파일 없음
```

Ask the user:
> "이슈로 등록할 항목을 선택하세요. 옵션: [a]ll, [r]eject all, [q]uit, or [s]elect (번호 쉼표 구분 예: s1,2,3):"

Wait for their response:
- `a` or Enter → approve all
- `r` → reject all and stop
- `q` → cancel and stop
- `s1,2,3` → approve only items 1, 2, 3

## Step 6: Create Linear Issues

For each approved finding:

1. Create a parent issue "📊 Repository Analysis Report" (only once, reuse if already exists)
2. For each approved finding, create a child issue under the parent with:
   - **Title**: "{finding.title}"
   - **Description**: "{finding.description}"
   - **Priority**: `P0` or `P1` (for 🔴), `P2` or `P3` (for 🟡)
   - **Label**: "{finding.category}"

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { issueCreate(input: {title: \"TITLE\", description: \"DESC\", teamId: \"TEAM_ID\", parentId: \"PARENT_ID\", priority: PRIORITY}) { issue { identifier title } } }"}'
```

For each created issue, capture the `identifier`. Tell the user:
```
이슈 생성됨:
  - ADE-100  README 파일 없음
  - ADE-101  테스트 디렉토리 없음
  - ADE-102  TODO/FIXME 주석 발견
```

## Step 7: Save Scan Results

Write `.claude/scan-results.json` with:

```json
{
  "scan_date": "2026-04-09T12:34:56Z",
  "scanned_by": "{USER_NAME}",
  "findings": [
    {
      "id": "finding_1",
      "category": "Documentation",
      "title": "README 파일 없음",
      "severity": "P1",
      "status": "created",
      "linear_issue": "ADE-100"
    }
  ],
  "summary": {
    "total": 4,
    "approved": 3,
    "created": 3
  }
}
```

If the file already exists, append to its history array instead of overwriting.

Tell the user:
```
✅ 스캔 완료

분석 결과:
  - 총 4개 발견
  - 3개 이슈 생성
  
다음 단계: /ls:start로 작업을 시작하세요.
```
