# /ls:setup — Linear 초기 설정

Setup begins now. Execute the following steps in sequence to configure Linear API integration.

**이 명령을 시작합니다.** 다음 단계를 순서대로 실행하세요.

## Step 1: Check for Existing API Key

Run this command to check if you have a Linear API key stored:

```bash
export PYTHONIOENCODING=utf-8
API_KEY="${LINEAR_API_KEY}"
if [ -z "$API_KEY" ]; then
  API_KEY=$(python3 -c "
import json, os
p = os.path.expanduser('~/.config/linear/config.json')
if os.path.exists(p):
    print(json.load(open(p)).get('api_key', ''))
" 2>/dev/null || echo "")
fi
echo "EXISTING_KEY=${API_KEY}"
```

If the output shows `EXISTING_KEY=` (empty), proceed to ask the user for their Linear API key. Otherwise, use the existing key.

**Ask the user if needed:**
> "Linear API 키가 필요합니다. Linear 웹 → Settings → API → Personal API Keys에서 생성하신 후 여기 붙여넣어주세요:"

Store the user's input (or the existing key) in `$API_KEY`.

## Step 2: Fetch Teams from Linear

Run this command to get the team list:

```bash
python3 << 'EOF'
import json
query = {'query': '{ teams { nodes { id name key } } }'}
print(json.dumps(query))
EOF
```

Save that JSON to a temporary file, then run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/linear_query.json
```

Parse the response. **If the JSON contains `"errors"` and mentions `401` or `Unauthorized`:**
- Tell the user: "API 키가 유효하지 않습니다. 다시 입력해주세요."
- Go back to Step 1.

**If there are other errors:**
- Tell the user the error message and stop.

**If successful**, extract the teams from the response. Display them as a numbered list:

```
1) Adevi (ADE)
2) Backend (BKD)
```

**Ask the user:**
> "팀을 선택하세요 (번호):"

Store the selected team's `id` → `$TEAM_ID` and `key` → `$TEAM_KEY`.

## Step 3: Fetch and Map Workflow States

Run this command to get the workflow states for the selected team:

```bash
python3 << 'EOF'
import json
import sys
team_id = "$TEAM_ID"
query = {
    'query': '''{ workflowStates(filter: { team: { id: { eq: "%s" } } }) { nodes { id name type } } }''' % team_id
}
print(json.dumps(query))
EOF
```

Save that JSON to a temporary file, then run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/linear_query.json
```

Parse the response and extract the workflow states. Display them as a numbered list with their names and types:

```
1) Backlog   (backlog)
2) Todo      (unstarted)
3) In Progress (started)
4) In Review   (started)
5) Done        (completed)
6) Canceled    (cancelled)
7) Duplicate   (cancelled)
```

**Ask the user for the role mappings:**

Present four prompts, one at a time:
> "작업 중 상태의 번호를 입력하세요 (예: 3):"
> "검토 중 상태의 번호를 입력하세요 (예: 4):"
> "완료 상태의 번호를 입력하세요 (예: 5):"
> "취소/중복 상태의 번호를 입력하세요 (예: 6):"

Extract each selected state's `id` and `name`. Store them as:
- `$IN_PROGRESS_ID` / `$IN_PROGRESS_NAME`
- `$IN_REVIEW_ID` / `$IN_REVIEW_NAME`
- `$DONE_ID` / `$DONE_NAME`
- `$CANCELED_ID` / `$CANCELED_NAME`

## Step 4: Detect Base Branch

Run:

```bash
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
echo "BASE_BRANCH=$BASE_BRANCH"
```

If `BASE_BRANCH` is empty, **ask the user:**
> "기준 브랜치를 입력하세요 (예: main, master):"

Store the user's input (or detected branch) in `$BASE_BRANCH`.

## Step 5: Check Admin Role and Scan Permission

Run this command to check if the user is an admin in the selected team:

```bash
python3 << 'EOF'
import json
import subprocess

team_id = "$TEAM_ID"
api_key = "$API_KEY"

query = {
    'query': '''{ viewer { 
      name 
      teamMembership(teamId: "%s") { role } 
    } }''' % team_id
}

# Write query to temp file
with open('/tmp/linear_query.json', 'w') as f:
    json.dump(query, f)

# Call curl
result = subprocess.run(
    ['curl', '-s', '-X', 'POST', 'https://api.linear.app/graphql',
     '-H', f'Authorization: {api_key}',
     '-H', 'Content-Type: application/json',
     '-d', '@/tmp/linear_query.json'],
    capture_output=True, text=True
)

response = json.loads(result.stdout)
viewer = response.get('data', {}).get('viewer', {})
membership = viewer.get('teamMembership', {})
user_name = viewer.get('name', 'Unknown')
role = membership.get('role', 'member')

print(f"USER_NAME={user_name}")
print(f"ROLE={role}")
EOF
```

Parse the output to extract `USER_NAME` and `ROLE`.

Tell the user:
> "Linear에서 당신의 역할: {ROLE} ({USER_NAME})"

**If the role is `admin`:**

Ask the user:
> "이 팀에서 저장소 스캔 기능(/ls:scan)을 활성화하시겠습니까? (Y/N):"

- If the user answers `Y` or `y`, set `$ENABLE_SCAN="true"` and tell them: "✅ 스캔 권한 활성화됨"
- Otherwise, set `$ENABLE_SCAN="false"` and tell them: "⚠️  스캔 권한 비활성화"

**If the role is not `admin`:**

Set `$ENABLE_SCAN="false"` and tell the user:
> "⚠️  주의: 스캔 기능은 팀장만 활성화할 수 있습니다. 현재 역할: {ROLE} (팀장 필요)"

## Step 6: Save Configuration Files

Create or update `~/.config/linear/config.json` with:

```json
{
  "api_key": "{API_KEY}"
}
```

Use the Write tool to create this file at the path `~/.config/linear/config.json` (expand `~` to the user's home directory).

Create `.claude/linear.json` in the current project directory:

```json
{
  "team_id": "{TEAM_ID}",
  "team_key": "{TEAM_KEY}",
  "base_branch": "{BASE_BRANCH}",
  "state_mapping": {
    "in_progress": {
      "id": "{IN_PROGRESS_ID}",
      "name": "{IN_PROGRESS_NAME}"
    },
    "in_review": {
      "id": "{IN_REVIEW_ID}",
      "name": "{IN_REVIEW_NAME}"
    },
    "done": {
      "id": "{DONE_ID}",
      "name": "{DONE_NAME}"
    },
    "canceled": {
      "id": "{CANCELED_ID}",
      "name": "{CANCELED_NAME}"
    }
  },
  "enable_scan": {ENABLE_SCAN},
  "scan_admin_user": "{USER_NAME}"
}
```

Use the Write tool to create `.claude/linear.json` at path `./.claude/linear.json` (relative to the current project root).

Tell the user:
```
✅ 설정 완료!

팀: {TEAM_KEY}
기준 브랜치: {BASE_BRANCH}
스캔 권한: {ENABLE_SCAN}

사용 가능한 커맨드:
  /ls:list       — 이슈 목록
  /ls:start      — 작업 시작
  /ls:pr         — PR 생성
  /ls:done       — 이슈 완료
  /ls:status     — 현재 상태
  /ls:sub        — 서브이슈 등록
  /ls:integrate  — 서브이슈 통합
```

If `{ENABLE_SCAN}` is `true`, also add:
```
  /ls:scan    — 레포지토리 분석 (팀장만)
```
