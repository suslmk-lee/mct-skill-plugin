# /ls:start [ISSUE-KEY] — 작업 세션 시작

Start command begins now. Execute the following steps to load an issue and create a working branch.

**이 명령을 시작합니다.**

## Step 1: Load Configuration

Check for the Linear API key:

```bash
API_KEY="${LINEAR_API_KEY}"
if [ -z "$API_KEY" ]; then
  API_KEY=$(python3 -c "
import json, os
p = os.path.expanduser('~/.config/linear/config.json')
if os.path.exists(p):
    print(json.load(open(p)).get('api_key', ''))
" 2>/dev/null || echo "")
fi
echo "API_KEY_SET=${#API_KEY}"
```

If `API_KEY` is empty, tell the user: "Linear API 키가 설정되지 않았습니다. 먼저 /ls:setup을 실행하세요."

Read `.claude/linear.json` and extract:
- `team_key`
- `base_branch`
- `state_mapping.in_progress.id` → `$IN_PROGRESS_ID`

## Step 2: Issue Selection

If no `[ISSUE-KEY]` argument was provided, ask the user to pick from their assigned issues:

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ viewer { id name } }"}'
```

Parse the response and extract `viewer.id` → `$VIEWER_ID`.

Then run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(first: 20, filter: {assignee: {id: {eq: \"VIEWER_ID\"}}, state: {type: {in: [\"backlog\", \"unstarted\"]}}}) { nodes { identifier title branchName state { name } } } }"}'
```

Parse the response. Extract the `issues.nodes` array. Display as a numbered list:

```
1) ADE-24  Backlog  로그 레벨 분리
2) ADE-25  Todo     워크플로우 히스토리 UI
```

**Ask the user:**
> "시작할 이슈 번호를 선택하세요:"

Store the selected issue's `identifier` in `$ISSUE_KEY`.

## Step 3: Fetch Detailed Issue Info

Extract team and number from `$ISSUE_KEY` (e.g., `ADE-24` → team=`ade`, number=`24`).

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(filter: {team: {key: {eq: \"TEAM\"}}, number: {eq: NUMBER}}) { nodes { id identifier title description branchName parent { id identifier branchName } state { id name } comments(last: 5) { nodes { body user { name } } } } } }"}'
```

Parse the response and extract:
- `issues.nodes[0].id` → `$ISSUE_UUID`
- `issues.nodes[0].title` → `$ISSUE_TITLE`
- `issues.nodes[0].description` → `$ISSUE_DESC`
- `issues.nodes[0].branchName` → `$BRANCH_NAME`
- `issues.nodes[0].parent.identifier` → `$PARENT_KEY` (may be empty)
- `issues.nodes[0].parent.branchName` → `$PARENT_BRANCH` (may be empty)
- Extract acceptance criteria: lines matching `- [ ]` pattern from `$ISSUE_DESC`
- Extract comments: `issues.nodes[0].comments.nodes[]`

If no issue is found, tell the user: "이슈를 찾을 수 없습니다: {ISSUE_KEY}" and stop.

## Step 4: Check for Dirty Working Tree

Run:

```bash
git status --porcelain
```

If the output is not empty (uncommitted changes exist), ask the user:
> "커밋하지 않은 변경사항이 있습니다. 계속 진행하시겠습니까? (Y/N):"

If the user answers `N` or `n`, stop.

## Step 5: Create or Switch Branch

If this is a sub-issue (`$PARENT_KEY` is not empty) AND the branch does not yet exist, ask the user:

```
이 이슈는 [{PARENT_KEY}]의 서브이슈입니다. 분기 기준 브랜치를 선택하세요:
  1) 현재 브랜치
  2) 부모 이슈 브랜치: {PARENT_BRANCH}
  3) 기본 브랜치: {BASE_BRANCH}
선택 (기본값 1):
```

Based on the user's choice, set `$BRANCH_BASE`. Otherwise, use `$BASE_BRANCH`.

Then run:

```bash
git show-ref --verify --quiet "refs/heads/{BRANCH_NAME}"
```

- If the branch exists, run: `git checkout {BRANCH_NAME}` and tell the user: "기존 브랜치로 전환: {BRANCH_NAME}"
- If the branch doesn't exist, run: `git checkout -b {BRANCH_NAME} {BRANCH_BASE}` and tell the user: "새 브랜치 생성: {BRANCH_NAME}"

## Step 6: Update Issue State to In Progress

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { issueUpdate(id: \"ISSUE_UUID\", input: { stateId: \"IN_PROGRESS_ID\" }) { issue { identifier state { name } } } }"}'
```

If successful, tell the user: "{ISSUE_KEY} 상태가 In Progress로 변경되었습니다."

## Step 7: Emit Session Context Block

Display the following context block. Extract acceptance criteria from `$ISSUE_DESC` (lines matching `- [ ]`). Include optional sections only if relevant:

```
## 현재 작업 세션
이슈: {ISSUE_KEY} — {ISSUE_TITLE}
부모 이슈: {PARENT_KEY}                 ← only if parent exists
설명: {ISSUE_DESC first 2 lines}
수락기준:                            ← only if acceptance criteria exist
  [ ] ...
  [ ] ...
최근 댓글: @{name} "{body}"           ← only if comments exist
브랜치: {BRANCH_NAME}
```

Tell the user: "작업을 시작합니다. 이 컨텍스트를 참조하여 진행하세요."
