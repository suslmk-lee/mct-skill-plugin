# /ls:list — 이슈 목록 조회

팀의 활성 이슈 목록을 조회합니다. 나에게 할당된 이슈를 먼저 표시합니다.

## Step 1: 설정 로드

```bash
export PYTHONIOENCODING=utf-8
API_KEY="$LINEAR_API_KEY"
if [ -z "$API_KEY" ]; then
  API_KEY=$(python3 -c "
import json, os
p = os.path.expanduser('~/.config/linear/config.json')
if os.path.exists(p): print(json.load(open(p)).get('api_key', ''))
" 2>/dev/null)
fi
if [ -z "$API_KEY" ]; then echo "API 키 없음. /ls:setup을 먼저 실행하세요"; exit 1; fi
if [ ! -f ".claude/linear.json" ]; then echo "프로젝트 설정 없음. /ls:setup을 먼저 실행하세요"; exit 1; fi
TEAM_KEY=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['team_key'].lower())")
```

## Step 2: 현재 사용자 + 브랜치 확인

```bash
# 현재 사용자 ID
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "import json; print(json.dumps({'query': '{ viewer { id name } }'}))" > "$TMPFILE"
VIEWER_RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
VIEWER_ID=$(echo "$VIEWER_RESPONSE" | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['viewer']['id'])")

# 현재 브랜치에서 이슈 키 추출 (매칭용)
CURRENT_BRANCH=$(git branch --show-current 2>/dev/null)
CURRENT_ISSUE=$(echo "$CURRENT_BRANCH" | grep -oP '^[a-z]+-\d+' | tr '[:lower:]' '[:upper:]' 2>/dev/null)
```

## Step 3: 이슈 목록 조회 (두 번 — 내 이슈 우선)

```bash
# 내 이슈 조회
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
team, viewer = sys.argv[1], sys.argv[2]
q = '''{ issues(first: 30, filter: {
  team: { key: { eq: \"''' + team + '''\" } },
  assignee: { id: { eq: \"''' + viewer + '''\" } },
  state: { type: { nin: [\"completed\", \"cancelled\"] } }
}, orderBy: updatedAt) {
  nodes { identifier title branchName state { name type } assignee { name } }
} }'''
print(json.dumps({'query': q}))
" "$TEAM_KEY" "$VIEWER_ID" > "$TMPFILE"
MY_ISSUES=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"

# 팀 전체 이슈 조회 (내 이슈 제외하여 병합)
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
team = sys.argv[1]
q = '''{ issues(first: 30, filter: {
  team: { key: { eq: \"''' + team + '''\" } },
  state: { type: { nin: [\"completed\", \"cancelled\"] } }
}, orderBy: updatedAt) {
  nodes { identifier title branchName state { name type } assignee { name } }
} }'''
print(json.dumps({'query': q}))
" "$TEAM_KEY" > "$TMPFILE"
ALL_ISSUES=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$MY_ISSUES"
echo "$ALL_ISSUES"
```

## Step 4: 결과 출력

두 응답을 병합하여 (내 이슈 먼저, 중복 제거) 다음 형식으로 출력합니다:

현재 브랜치의 이슈(`CURRENT_ISSUE`)가 있으면 해당 행 앞에 `★`를 표시합니다.
담당자 표시 규칙:
- `assignee.id == VIEWER_ID`이면 `(나)` 표시
- `assignee`가 있고 다른 사람이면 `(assignee.name)` 표시
- `assignee`가 없으면 `-` 표시

```
★ ADE-24  In Progress  ade-24-log-level-separation       (나)
  ADE-25  Todo         ade-25-workflow-history-ui         (나)
  ADE-26  Backlog      ade-26-payment-refactor            (kim)
  ADE-27  Backlog      ade-27-auth-refresh                -
```
