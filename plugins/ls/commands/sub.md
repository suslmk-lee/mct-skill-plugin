# /ls:sub [PARENT-KEY] — 서브이슈 등록

You are executing `/ls:sub [PARENT-KEY]`. Follow these steps in order. This creates one or more sub-issues under a parent issue.

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
- `team_id`
- `team_key`

## Step 2: Resolve Parent Issue

If no `[PARENT-KEY]` argument was provided, try to extract it from the current Git branch:

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
PARENT_KEY=$(echo "$CURRENT_BRANCH" | grep -oE '^[a-z]+-[0-9]+' || echo "")
echo "PARENT_KEY=$PARENT_KEY"
```

If `PARENT_KEY` is still empty, ask the user:
> "부모 이슈 키를 입력하세요 (예: ADE-24):"

Extract team and number from `$PARENT_KEY` (e.g., `ADE-24` → team=`ade`, number=`24`).

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(filter: {team: {key: {eq: \"TEAM\"}}, number: {eq: NUMBER}}) { nodes { id identifier title children { nodes { identifier title state { name } } } } } }"}'
```

Parse the response and extract:
- `issues.nodes[0].id` → `$PARENT_UUID`
- `issues.nodes[0].identifier` → `$PARENT_KEY_RESOLVED`
- `issues.nodes[0].title` → `$PARENT_TITLE`
- `issues.nodes[0].children.nodes[]` → existing sub-issues array

If no issue is found, tell the user: "이슈를 찾을 수 없습니다: {PARENT_KEY}" and stop.

If existing sub-issues are found, display them:

```
부모 이슈: ADE-24 — 로그 레벨 분리

기존 서브이슈:
  ADE-28  Todo  로그 레벨 enum 정의
  ADE-29  Todo  환경변수 로딩 모듈
```

## Step 3: Collect Sub-Issue Titles

Ask the user to provide sub-issue titles:

> "추가할 서브이슈를 입력하세요 (한 줄에 하나, 빈 줄로 완료):"

Wait for the user to enter multiple lines. Stop when they enter a blank line. Store each title in a list.

If the user provided at least one sub-issue, ask:
> "각 서브이슈에 설명을 추가할까요? (Y/N):"

If they answer yes, ask for a description for each sub-issue (optional).

## Step 4: Create Sub-Issues

For each sub-issue title collected in Step 3, run the Linear GraphQL mutation:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { issueCreate(input: {title: \"TITLE\", description: \"DESC\", teamId: \"TEAM_ID\", parentId: \"PARENT_UUID\"}) { issue { identifier title } } }"}'
```

Parse each response and extract:
- `issueCreate.issue.identifier` → new sub-issue key
- `issueCreate.issue.title` → new sub-issue title

Collect all created sub-issues in a list.

## Step 5: Display Completion Summary

Tell the user:

```
서브이슈 등록 완료:

{PARENT_KEY} — {PARENT_TITLE}
  ├─ {SUB_KEY_1}  {SUB_TITLE_1}
  └─ {SUB_KEY_2}  {SUB_TITLE_2}

작업을 시작하려면: /ls:start {SUB_KEY_1}
```
