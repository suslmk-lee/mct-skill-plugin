# /ls:pr — PR 생성

현재 브랜치의 변경사항으로 PR을 생성합니다.

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
BASE_BRANCH=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['base_branch'])")
IN_REVIEW_ID=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['state_mapping']['in_review']['id'])")
```

## Step 2: gh CLI 확인

```bash
which gh || { echo "gh CLI가 필요합니다: https://cli.github.com"; exit 1; }
gh auth status || { echo "gh 인증이 필요합니다: gh auth login"; exit 1; }
```

## Step 3: 이슈 키 추출 + 이슈 정보 로드

```bash
CURRENT_BRANCH=$(git branch --show-current)
ISSUE_KEY=$(echo "$CURRENT_BRANCH" | grep -oP '^[a-z]+-\d+' | tr '[:lower:]' '[:upper:]')
if [ -z "$ISSUE_KEY" ]; then
  echo "이슈 키를 입력하세요 (예: ADE-24):"
fi
```

세션 컨텍스트 블록이 있으면 그것을 사용합니다.
없으면 이슈를 재조회합니다:

```bash
ISSUE_TEAM=$(echo "$ISSUE_KEY" | tr '[:upper:]' '[:lower:]' | cut -d'-' -f1)
ISSUE_NUM=$(echo "$ISSUE_KEY" | cut -d'-' -f2)

TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
team, num = sys.argv[1], int(sys.argv[2])
q = '''{ issues(filter: {
  team: { key: { eq: \"''' + team + '''\" } },
  number: { eq: ''' + str(num) + ''' }
}) { nodes {
  id identifier title description
  comments(last: 5) { nodes { body user { name } } }
  parent { id identifier }
  children { nodes { identifier title state { name type } } }
} } }'''
print(json.dumps({'query': q}))
" "$ISSUE_TEAM" "$ISSUE_NUM" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"

ISSUE_UUID=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
print(nodes[0]['id'] if nodes else '')
")
ISSUE_TITLE=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
print(nodes[0]['title'] if nodes else '')
")
if [ -z "$ISSUE_UUID" ]; then echo "이슈를 찾을 수 없습니다: $ISSUE_KEY"; exit 1; fi
```

## Step 3a: 이슈 유형 판단

```bash
PARENT_KEY=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
parent = nodes[0].get('parent') if nodes else None
print(parent['identifier'] if parent else '')
")

CHILDREN_COUNT=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
children = nodes[0].get('children', {}).get('nodes', []) if nodes else []
print(len(children))
")

if [ -n "$PARENT_KEY" ]; then
  echo "경고: 이 이슈는 [$PARENT_KEY]의 서브이슈입니다."
  echo "서브이슈는 일반적으로 PR을 생성하지 않습니다."
  echo "계속 진행하시겠습니까? (y/n):"
  # n이면 중단
fi
```

## Step 4: diff 분석

```bash
git diff $BASE_BRANCH...HEAD --stat
git diff $BASE_BRANCH...HEAD
```

## Step 5: PR 초안 생성

위에서 수집한 정보를 기반으로 PR 초안을 작성합니다:

```bash
if [ "$CHILDREN_COUNT" -gt 0 ]; then
  SUB_SUMMARY=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
children = nodes[0].get('children', {}).get('nodes', []) if nodes else []
lines = []
for c in children:
    done = c['state']['type'] in ('completed', 'cancelled')
    mark = 'x' if done else ' '
    lines.append(f'- [{mark}] {c[\"identifier\"]} \u2014 {c[\"title\"]}')
print('\n'.join(lines))
")
fi
```

- **제목**: `[{ISSUE_KEY}] {ISSUE_TITLE}`
- **본문**:

```markdown
## 개요
{이슈 설명 기반 요약 — 1-3 문장}

## 서브이슈 완료 목록
{SUB_SUMMARY — CHILDREN_COUNT > 0 일 때만 이 섹션 포함}

## 변경 사항
{git diff 분석 기반 bullet points}

## 수락기준
{세션 컨텍스트의 수락기준 항목 — 달성 여부 판단해서 [x]/[ ] 표시}
(수락기준 없으면 이 섹션 생략)

## Linear 이슈
https://linear.app/issue/{ISSUE_KEY}

🤖 Generated with Claude Code
```

초안을 사용자에게 보여주고 확인을 요청합니다:
> "이 내용으로 PR을 생성할까요? (y/수정 요청):"

수정 요청이 있으면 수정 후 다시 확인을 요청합니다.

## Step 6: PR 생성

사용자가 확인하면:

```bash
PR_TITLE="[$ISSUE_KEY] $ISSUE_TITLE"

BODY_FILE=$(mktemp /tmp/pr-body-XXXXXX.md)
cat > "$BODY_FILE" << 'BODY_EOF'
{위에서 작성한 PR 본문 내용}
BODY_EOF

PR_URL=$(gh pr create \
  --title "$PR_TITLE" \
  --body-file "$BODY_FILE" \
  --base "$BASE_BRANCH")
rm -f "$BODY_FILE"
echo "PR 생성됨: $PR_URL"
```

PR 생성 실패 시 gh 에러 메시지를 그대로 출력하고 중단합니다.

## Step 7: 이슈 상태 → In Review

```bash
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
mutation = '''mutation {
  issueUpdate(id: \"''' + sys.argv[1] + '''\", input: { stateId: \"''' + sys.argv[2] + '''\" }) {
    issue { identifier state { name } }
  }
}'''
print(json.dumps({'query': mutation}))
" "$ISSUE_UUID" "$IN_REVIEW_ID" > "$TMPFILE"
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE" > /dev/null
rm -f "$TMPFILE"
```
