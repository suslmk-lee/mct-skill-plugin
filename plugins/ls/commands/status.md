# /ls:status — 현재 작업 스냅샷

Status check starts now. Execute the following steps sequentially.

**이 명령을 시작합니다.**

## Step 1: Extract Issue Key from Current Branch

Run:

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
ISSUE_KEY=$(echo "$CURRENT_BRANCH" | grep -oE '^[a-z]+-[0-9]+' || echo "")
echo "ISSUE_KEY=$ISSUE_KEY"
```

If `ISSUE_KEY` is empty, tell the user: "Linear 이슈와 연결된 브랜치가 아닙니다. (현재 브랜치: {CURRENT_BRANCH})" and stop.

## Step 2: Load Configuration

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

## Step 3: Fetch Issue Status

Extract the team key and issue number from `$ISSUE_KEY` (e.g., `ADE-24` → team=`ade`, number=`24`).

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(filter: {team: {key: {eq: \"TEAM_KEY\"}}, number: {eq: ISSUE_NUMBER}}) { nodes { identifier title state { name } } } }"}'
```

Parse the response and extract:
- `issues.nodes[0].identifier` → `$ISSUE_KEY_RESOLVED`
- `issues.nodes[0].title` → `$ISSUE_TITLE`
- `issues.nodes[0].state.name` → `$ISSUE_STATE`

If no issue is found, tell the user: "이슈를 찾을 수 없습니다." and stop.

## Step 4: Check Git Status

Run:

```bash
git status --short
```

Count the number of lines. If 0, set `$GIT_STATUS_TEXT="없음"`. Otherwise, set it to the count (e.g., "3 files").

## Step 5: Display Status

Tell the user:
```
현재 작업: {ISSUE_KEY} — {ISSUE_TITLE}
상태: {ISSUE_STATE}
브랜치: {CURRENT_BRANCH}
미커밋 변경: {GIT_STATUS_TEXT}
```
