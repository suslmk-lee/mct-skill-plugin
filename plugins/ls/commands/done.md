# /ls:done [ISSUE-KEY] — 이슈 완료

Mark issue as done starts now. Execute the following steps to complete an issue.

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
- `state_mapping.done.id` → `$DONE_ID`
- `state_mapping.done.name` → `$DONE_NAME`
- `state_mapping.canceled.name` → `$CANCELED_NAME`

## Step 2: Resolve Issue Key

If no `[ISSUE-KEY]` argument was provided, try to extract it from the current Git branch:

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
ISSUE_KEY=$(echo "$CURRENT_BRANCH" | grep -oE '^[a-z]+-[0-9]+' || echo "")
echo "ISSUE_KEY=$ISSUE_KEY"
```

If `ISSUE_KEY` is still empty, ask the user:
> "이슈 키를 입력하세요 (예: ADE-24):"

## Step 3: Fetch Issue Details

Extract team and number from `$ISSUE_KEY`. Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(filter: {team: {key: {eq: \"TEAM\"}}, number: {eq: NUMBER}}) { nodes { id identifier title description state { name type } parent { id identifier } children { nodes { identifier state { type } } } } } }"}'
```

Parse the response and extract:
- `issues.nodes[0].id` → `$ISSUE_UUID`
- `issues.nodes[0].title` → `$ISSUE_TITLE`
- `issues.nodes[0].description` → `$ISSUE_DESC`
- `issues.nodes[0].state.name` → `$CURRENT_STATE`
- `issues.nodes[0].parent` → `$PARENT_INFO` (may be empty)
- `issues.nodes[0].children.nodes[]` → sub-issues array

If no issue is found, tell the user: "이슈를 찾을 수 없습니다: {ISSUE_KEY}" and stop.

## Step 4: Check Acceptance Criteria (Optional)

If there's a session context block, extract acceptance criteria (lines matching `- [ ]`). If criteria exist, check the current codebase to determine which are met. Display a checklist:

```
수락기준:
  [x] ...
  [ ] ...
```

Ask the user:
> "모든 수락기준이 충족되었나요? (Y/N):"

If they answer `N`, ask:
> "계속 진행하시겠습니까? (Y/N):"

If they answer `N`, stop.

## Step 5: Update Issue State to Done

Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { issueUpdate(id: \"ISSUE_UUID\", input: { stateId: \"DONE_ID\" }) { issue { identifier state { name } } } }"}'
```

If successful, tell the user: "{ISSUE_KEY} — {ISSUE_TITLE} / 상태: Done ✓"

## Step 6: Check Parent Issue Sub-Issues

If `$PARENT_INFO` is empty (this is not a sub-issue), tell the user: "완료된 이슈입니다." and stop.

If this is a sub-issue, check all siblings:
- If all sub-issues are done or canceled, ask the user:
  > "모든 서브이슈가 완료되었습니다. `/ls:integrate [PARENT_KEY]` 를 실행하여 통합하시겠습니까?"
- If some sub-issues are still pending, tell the user:
  > "현재 {PARENT_KEY}의 {REMAINING_COUNT}개 서브이슈가 대기 중입니다."
