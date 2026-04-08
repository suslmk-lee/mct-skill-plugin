# /ls:integrate [PARENT-KEY] — 서브이슈 통합 및 부모 PR 생성

서브이슈 브랜치 체인을 부모 이슈 브랜치에 머지하고 PR을 생성합니다.

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

## Step 2: 부모 이슈 키 확인

PARENT-KEY 인자가 없으면 현재 브랜치에서 추출합니다:

```bash
if [ -z "$PARENT_KEY" ]; then
  CURRENT_BRANCH=$(git branch --show-current 2>/dev/null)
  PARENT_KEY=$(echo "$CURRENT_BRANCH" | grep -oE '^[a-zA-Z]+-[0-9]+' | tr '[:lower:]' '[:upper:]')
fi
if [ -z "$PARENT_KEY" ]; then
  echo "부모 이슈 키를 입력하세요 (예: ADE-20):"
  # 사용자 입력 대기
fi
```

## Step 3: 부모 이슈 + 서브이슈 목록 조회

```bash
PARENT_TEAM=$(echo "$PARENT_KEY" | tr '[:upper:]' '[:lower:]' | cut -d'-' -f1)
PARENT_NUM=$(echo "$PARENT_KEY" | cut -d'-' -f2)

TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
team, num = sys.argv[1], int(sys.argv[2])
q = '''{ issues(filter: {
  team: { key: { eq: \"''' + team + '''\" } },
  number: { eq: ''' + str(num) + ''' }
}) { nodes {
  id identifier title description branchName
  state { name type }
  children { nodes { identifier title branchName state { name type } } }
  comments(last: 5) { nodes { body user { name } } }
} } }'''
print(json.dumps({'query': q}))
" "$PARENT_TEAM" "$PARENT_NUM" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"

PARENT_UUID=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
print(nodes[0]['id'] if nodes else '')
")
PARENT_TITLE=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
print(nodes[0]['title'] if nodes else '')
")
PARENT_BRANCH=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
print(nodes[0]['branchName'] if nodes else '')
")
if [ -z "$PARENT_UUID" ]; then echo "이슈를 찾을 수 없습니다: $PARENT_KEY"; exit 1; fi
```

서브이슈 목록과 완료 현황을 표시합니다:

```
ADE-20 — RBAC 역할 기반 접근 제어 구현
브랜치: mctlmk/ade-20

서브이슈:
  [x] ADE-39  Done   DB 모델 정의              (mctlmk/ade-39)
  [x] ADE-40  Done   JWT 유틸 구현             (mctlmk/ade-40)
  [x] ADE-41  Done   인증 API                  (mctlmk/ade-41)
  [ ] ADE-42  Todo   권한 미들웨어              (mctlmk/ade-42)
```

완료되지 않은 서브이슈가 있으면:
> "미완료 서브이슈가 있습니다. 완료된 것만 통합할까요? (y/n):"

n이면 중단합니다.

## Step 4: 통합할 브랜치 결정

서브이슈 브랜치 중 **마지막으로 완료된 것**을 찾습니다.
브랜치가 체인 구조이므로 마지막 브랜치를 머지하면 이전 브랜치의 변경사항이 모두 포함됩니다.

```bash
LAST_SUB_BRANCH=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
children = nodes[0].get('children', {}).get('nodes', []) if nodes else []
done_children = [c for c in children if c['state']['type'] in ('completed', 'cancelled')]
print(done_children[-1]['branchName'] if done_children else '')
")
if [ -z "$LAST_SUB_BRANCH" ]; then
  echo "완료된 서브이슈가 없습니다."; exit 1
fi
echo "통합 대상: $LAST_SUB_BRANCH → $PARENT_BRANCH"
echo "계속 진행하시겠습니까? (y/n):"
# n이면 중단
```

## Step 5: 부모 브랜치 체크아웃 및 머지

```bash
# 부모 브랜치가 로컬에 없으면 생성
if ! git show-ref --verify --quiet "refs/heads/$PARENT_BRANCH"; then
  git checkout -b "$PARENT_BRANCH" "$BASE_BRANCH"
  echo "부모 브랜치 생성: $PARENT_BRANCH"
else
  git checkout "$PARENT_BRANCH"
fi

# 서브이슈 체인 머지
git merge --no-ff "$LAST_SUB_BRANCH" -m "integrate: merge sub-issue chain into $PARENT_BRANCH"
```

머지 충돌 시:
> "머지 충돌이 발생했습니다. 충돌을 해결한 후 git merge --continue를 실행하세요."

중단합니다.

## Step 6: gh CLI 확인

```bash
which gh || { echo "gh CLI가 필요합니다: https://cli.github.com"; exit 1; }
gh auth status || { echo "gh 인증이 필요합니다: gh auth login"; exit 1; }
```

## Step 7: PR 초안 생성

서브이슈 목록을 포함한 PR 본문을 작성합니다.

```bash
git diff "$BASE_BRANCH"..."$PARENT_BRANCH" --stat
git diff "$BASE_BRANCH"..."$PARENT_BRANCH"
```

- **제목**: `[{PARENT_KEY}] {PARENT_TITLE}`
- **본문**:

```markdown
## 개요
{PARENT_TITLE 및 이슈 설명 기반 1-3 문장}

## 서브이슈 완료 목록
{완료된 서브이슈 목록}
- [x] ADE-39 — DB 모델 정의
- [x] ADE-40 — JWT 유틸 구현
- [x] ADE-41 — 인증 API

## 변경 사항
{git diff BASE_BRANCH...PARENT_BRANCH 분석 기반 bullet points}

## Linear 이슈
https://linear.app/issue/{PARENT_KEY}

🤖 Generated with Claude Code
```

초안을 사용자에게 보여주고 확인 요청:
> "이 내용으로 PR을 생성할까요? (y/수정 요청):"

수정 요청이 있으면 수정 후 다시 확인을 요청합니다.

## Step 8: PR 생성 및 push

```bash
git push -u origin "$PARENT_BRANCH"

PR_TITLE="[$PARENT_KEY] $PARENT_TITLE"

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

## Step 9: 이슈 상태 → In Review

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
" "$PARENT_UUID" "$IN_REVIEW_ID" > "$TMPFILE"
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE" > /dev/null
rm -f "$TMPFILE"
```

## 완료 메시지

```
{PARENT_KEY} — {PARENT_TITLE}
브랜치: {PARENT_BRANCH}
PR: {PR_URL}
상태: In Review
```
