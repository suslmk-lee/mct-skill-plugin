# /ls:pr — PR 생성

PR creation starts now. Execute the following steps to create a GitHub pull request.

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
- `base_branch`
- `state_mapping.in_review.id` → `$IN_REVIEW_ID`

## Step 2: Verify GitHub CLI

Run:

```bash
which gh
gh auth status
```

If `gh` is not found or not authenticated, tell the user: "GitHub CLI가 필요합니다. https://cli.github.com 에서 설치 후 gh auth login 을 실행하세요."

## Step 3: Extract Issue Key and Load Issue Info

Run:

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
ISSUE_KEY=$(echo "$CURRENT_BRANCH" | grep -oE '^[a-z]+-[0-9]+' || echo "")
echo "ISSUE_KEY=$ISSUE_KEY"
```

If `ISSUE_KEY` is empty, ask the user:
> "이슈 키를 입력하세요 (예: ADE-24):"

Check if there's a session context block in the conversation. If it exists, use the issue information from it. Otherwise, fetch from Linear:

Extract team and number from `$ISSUE_KEY`. Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(filter: {team: {key: {eq: \"TEAM\"}}, number: {eq: NUMBER}}) { nodes { id identifier title description parent { id identifier } children { nodes { identifier title state { name type } } } } } }"}'
```

Parse the response and extract:
- `issues.nodes[0].id` → `$ISSUE_UUID`
- `issues.nodes[0].title` → `$ISSUE_TITLE`
- `issues.nodes[0].description` → `$ISSUE_DESC`
- `issues.nodes[0].parent.identifier` → `$PARENT_KEY` (may be empty)
- `issues.nodes[0].children.nodes[]` → sub-issues list

If no issue is found, tell the user: "이슈를 찾을 수 없습니다: {ISSUE_KEY}" and stop.

## Step 4: Check Issue Type

If `$PARENT_KEY` is not empty, tell the user:
```
경고: 이 이슈는 [{PARENT_KEY}]의 서브이슈입니다.
서브이슈는 일반적으로 PR을 생성하지 않습니다.
계속 진행하시겠습니까? (Y/N):
```

If the user answers `N`, stop.

## Step 5: Analyze Diff

Run:

```bash
git diff {BASE_BRANCH}...HEAD --stat
git diff {BASE_BRANCH}...HEAD
```

Capture the stat and full diff output. Use this to generate bullet points summarizing the changes.

## Step 6: Draft PR Body

Compose a PR body with these sections:

- **Title**: `[{ISSUE_KEY}] {ISSUE_TITLE}`
- **Body**:
  - **개요** — 1–3 sentences from `$ISSUE_DESC`
  - **서브이슈 완료 목록** (if sub-issues exist) — list each with `- [x]` or `- [ ]` based on completion
  - **변경 사항** — bullet points from diff analysis
  - **수락기준** (if criteria exist from session context) — list with `[x]` or `[ ]` marks
  - **Linear 이슈** — `https://linear.app/{ISSUE_KEY}`

Show the draft to the user:
> "다음 내용으로 PR을 생성할까요? (Y/수정 요청):"

If the user asks for revisions, accept their input and update the draft. Re-confirm.

## Step 7: Create PR on GitHub

Once confirmed, write the PR body to a temporary file and run:

```bash
gh pr create \
  --title "[{ISSUE_KEY}] {ISSUE_TITLE}" \
  --body-file {body_file} \
  --base {BASE_BRANCH}
```

Capture the PR URL from the output. Tell the user: "PR 생성됨: {PR_URL}"

If the command fails, tell the user the error message and stop.

## Step 8: Update Issue State to In Review

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { issueUpdate(id: \"ISSUE_UUID\", input: { stateId: \"IN_REVIEW_ID\" }) { issue { identifier state { name } } } }"}'
```

If successful, tell the user: "{ISSUE_KEY} 상태가 In Review로 변경되었습니다."
