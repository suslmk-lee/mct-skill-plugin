# /ls:status — 현재 작업 스냅샷

현재 브랜치에 연결된 Linear 이슈와 Git 상태를 표시합니다.

## Step 1: 브랜치에서 이슈 키 추출

```bash
export PYTHONIOENCODING=utf-8
CURRENT_BRANCH=$(git branch --show-current 2>/dev/null)
ISSUE_KEY=$(echo "$CURRENT_BRANCH" | grep -oP '^[a-z]+-\d+' | tr '[:lower:]' '[:upper:]')
if [ -z "$ISSUE_KEY" ]; then
  echo "Linear 이슈와 연결된 브랜치가 아닙니다. (현재 브랜치: $CURRENT_BRANCH)"
  exit 0
fi
```

## Step 2: 설정 로드

```bash
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
```

## Step 3: 이슈 상태 조회

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
}) { nodes { identifier title state { name } } } }'''
print(json.dumps({'query': q}))
" "$ISSUE_TEAM" "$ISSUE_NUM" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$RESPONSE"
```

## Step 4: Git 상태 확인

```bash
git status --short
```

## Step 5: 결과 출력

```
현재 작업: {ISSUE_KEY} — {ISSUE_TITLE}
상태: {ISSUE_STATE}
브랜치: {CURRENT_BRANCH}
미커밋 변경: {git status --short 줄 수} files
```

미커밋 변경이 없으면 "미커밋 변경: 없음"으로 표시합니다.
