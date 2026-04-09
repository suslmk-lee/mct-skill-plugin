# /ls:list — 이슈 목록 조회

You are executing `/ls:list`. Follow these steps in order.

## Step 1: Load Configuration

Read the `.claude/linear.json` file in the current project directory. Extract:
- `team_key`

Also check for the Linear API key from the environment or config:

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

Also detect the current issue key from the Git branch name by running:

```bash
CURRENT_ISSUE=$(git rev-parse --abbrev-ref HEAD 2>/dev/null | grep -oE '^[a-z]+-[0-9]+' || echo "")
echo "CURRENT_ISSUE=$CURRENT_ISSUE"
```

Store the current issue (may be empty) in `$CURRENT_ISSUE`.

## Step 2: Get Current User ID

Run:

```bash
python3 << 'EOF'
import json
query = {'query': '{ viewer { id name } }'}
print(json.dumps(query))
EOF
```

Call the Linear GraphQL API:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ viewer { id name } }"}'
```

Parse the response and extract `viewer.id` → `$VIEWER_ID`.

## Step 3: Fetch Issues (two queries)

**Query 1: User's assigned issues** (first 30, non-completed/cancelled):

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(first: 30, filter: {team: {key: {eq: \"TEAM_KEY\"}}, assignee: {id: {eq: \"VIEWER_ID\"}}, state: {type: {nin: [\"completed\", \"cancelled\"]}}}, orderBy: updatedAt) { nodes { identifier title branchName state { name } assignee { name } } } }"}'
```

Parse the response and extract the `issues.nodes` array as `$MY_ISSUES`.

**Query 2: All team issues** (first 30, non-completed/cancelled):

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(first: 30, filter: {team: {key: {eq: \"TEAM_KEY\"}}, state: {type: {nin: [\"completed\", \"cancelled\"]}}}, orderBy: updatedAt) { nodes { identifier title branchName state { name } assignee { name } } } }"}'
```

Parse the response and extract the `issues.nodes` array as `$ALL_ISSUES`.

## Step 4: Display Issues

Combine both results:
1. Start with user's assigned issues (`$MY_ISSUES`)
2. Add team issues (`$ALL_ISSUES`) that are not already in the user's list (deduplicate by `identifier`)

Display in a formatted table. For each issue:
- If the issue's `identifier` matches `$CURRENT_ISSUE`, prefix with `★` (star)
- Show: `identifier`, `state.name`, `branchName`, assignee
  - If assignee exists, show `(assignee.name)`
  - If no assignee, show `-`

Example output:
```
★ ADE-24  In Progress  ade-24-log-level-separation       (이름)
  ADE-25  Todo         ade-25-workflow-history-ui         (이름)
  ADE-26  Backlog      ade-26-payment-refactor            -
  ADE-27  Backlog      ade-27-auth-refresh                (다른사람)
```
