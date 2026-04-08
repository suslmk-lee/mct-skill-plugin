# /ls:done [ISSUE-KEY] — 이슈 완료

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
DONE_ID=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['state_mapping']['done']['id'])")
DONE_NAME=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['state_mapping']['done']['name'])")
CANCELED_NAME=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['state_mapping']['canceled']['name'])")
```

## Step 2: 이슈 키 확인

ISSUE-KEY 인자가 없으면 현재 브랜치에서 추출합니다:

```bash
if [ -z "$ISSUE_KEY" ]; then
  CURRENT_BRANCH=$(git branch --show-current 2>/dev/null)
  ISSUE_KEY=$(echo "$CURRENT_BRANCH" | grep -oP '^[a-z]+-\d+' | tr '[:lower:]' '[:upper:]')
fi
if [ -z "$ISSUE_KEY" ]; then
  echo "이슈 키를 입력하세요 (예: ADE-24):"
  # 사용자 입력 대기
fi
```

## Step 3: 이슈 UUID + 현재 상태 조회

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
  id identifier title description state { name type }
  parent { id identifier branchName
    children { nodes { identifier state { type } } }
  }
} } }'''
print(json.dumps({'query': q}))
" "$ISSUE_TEAM" "$ISSUE_NUM" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$RESPONSE"
```

응답에서:
- `ISSUE_UUID`, `ISSUE_TITLE`, `ISSUE_DESC`, `CURRENT_STATE_NAME`, `CURRENT_STATE_TYPE` 추출
- `PARENT_KEY`: `data.issues.nodes[0].parent.identifier`
- `PARENT_BRANCH`: `data.issues.nodes[0].parent.branchName`

현재 상태 유형 확인:
- `completed`: "이미 완료된 이슈입니다." 출력 후 종료
- `cancelled`: "취소된 이슈입니다. 그래도 Done으로 변경할까요? (y/n):" → n이면 종료

## Step 4: 수락기준 달성 여부 판단

세션 컨텍스트 블록이 있으면, 현재 코드베이스와 비교하여 각 수락기준 항목의 달성 여부를 판단합니다.
판단 결과를 다음 형식으로 출력합니다:

```
수락기준 점검:
  [x] 로그 레벨별 필터링 가능      ← 달성
  [ ] 기본 레벨은 환경변수로 설정 가능  ← 미달성 (코드에서 확인 필요)
```

미달성 항목이 있으면:
> "미완료 수락기준이 있습니다. 그래도 Done으로 변경할까요? (y/n):"

## Step 5: 이슈 상태 → Done

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
" "$ISSUE_UUID" "$DONE_ID" > "$TMPFILE"
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE" > /dev/null
rm -f "$TMPFILE"
```

## Step 6: 서브이슈 완료 체크

```bash
if [ -n "$PARENT_KEY" ]; then
  TOTAL=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
parent = nodes[0].get('parent') if nodes else None
children = parent.get('children', {}).get('nodes', []) if parent else []
print(len(children))
")
  DONE_COUNT=$(echo "$RESPONSE" | python3 -c "
import json, sys
nodes = json.load(sys.stdin)['data']['issues']['nodes']
parent = nodes[0].get('parent') if nodes else None
children = parent.get('children', {}).get('nodes', []) if parent else []
completed = [c for c in children if c['state']['type'] in ('completed', 'cancelled')]
print(len(completed))
")

  if [ "$DONE_COUNT" -eq "$TOTAL" ] && [ "$TOTAL" -gt 0 ]; then
    echo ""
    echo "모든 서브이슈가 완료되었습니다 ($DONE_COUNT/$TOTAL)"
    echo "부모 이슈 [$PARENT_KEY] 브랜치에 통합할 준비가 되었습니다."
    echo "  → /ls:integrate $PARENT_KEY 를 실행하세요"
  else
    REMAINING=$((TOTAL - DONE_COUNT))
    echo ""
    echo "서브이슈 진행: $DONE_COUNT/$TOTAL 완료 (미완료 ${REMAINING}개)"
  fi
fi
```

## Step 7: 완료 메시지

```
{ISSUE_KEY} — {ISSUE_TITLE}
상태: {DONE_NAME} ✓
```
