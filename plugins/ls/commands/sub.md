# /ls:sub [PARENT-KEY] — 서브이슈 등록

특정 이슈 하위에 서브이슈를 생성합니다.

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
TEAM_ID=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['team_id'])")
TEAM_KEY=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['team_key'])")
```

## Step 2: 부모 이슈 UUID 조회

PARENT-KEY 인자가 없으면 현재 브랜치에서 이슈 키를 추출합니다:

```bash
if [ -z "$PARENT_KEY" ]; then
  CURRENT_BRANCH=$(git branch --show-current 2>/dev/null)
  PARENT_KEY=$(echo "$CURRENT_BRANCH" | grep -oE '^[a-zA-Z]+-[0-9]+' | tr '[:lower:]' '[:upper:]')
fi
if [ -z "$PARENT_KEY" ]; then
  echo "부모 이슈 키를 입력하세요 (예: ADE-24):"
fi
```

부모 이슈 UUID + 상세 조회:

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
}) { nodes { id identifier title children { nodes { identifier title state { name } } } } } }'''
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
if [ -z "$PARENT_UUID" ]; then echo "이슈를 찾을 수 없습니다: $PARENT_KEY"; exit 1; fi
```

기존 서브이슈가 있으면 목록을 표시합니다:

```
부모 이슈: ADE-24 — 로그 레벨 분리

기존 서브이슈:
  ADE-28  Todo  로그 레벨 enum 정의
  ADE-29  Todo  환경변수 로딩 모듈
```

## Step 3: 서브이슈 입력

사용자에게 서브이슈 목록을 입력받습니다:

```
추가할 서브이슈를 입력하세요 (한 줄에 하나, 빈 줄로 완료):
> Sentry 연동 모듈 작성
> 로그 레벨별 필터 테스트 작성
>
```

각 항목에 대해 선택적으로 설명과 우선순위를 물어봅니다:
> "각 서브이슈에 설명을 추가할까요? (y/n):"

## Step 4: 서브이슈 등록

`parentId`를 지정하여 서브이슈를 생성합니다:

```bash
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
title  = sys.argv[1]
desc   = sys.argv[2]
team   = sys.argv[3]
parent = sys.argv[4]
mutation = '''mutation {
  issueCreate(input: {
    title: \"''' + title.replace('\"','\\\\\"') + '''\",
    description: \"''' + desc.replace('\"','\\\\\"') + '''\",
    teamId: \"''' + team + '''\",
    parentId: \"''' + parent + '''\"
  }) { issue { identifier title url } }
}'''
print(json.dumps({'query': mutation}, ensure_ascii=False))
" "$SUB_TITLE" "$SUB_DESC" "$TEAM_ID" "$PARENT_UUID" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$RESPONSE"
```

## Step 5: 완료 요약

```
서브이슈 등록 완료:

ADE-24 — 로그 레벨 분리
  ├─ ADE-30  Sentry 연동 모듈 작성
  └─ ADE-31  로그 레벨별 필터 테스트 작성

작업을 시작하려면: /ls:start ADE-30
```
