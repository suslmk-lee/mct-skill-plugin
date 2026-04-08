# /ls:setup — Linear 초기 설정

API 키 등록, 팀 선택, 워크플로우 상태 매핑을 진행합니다.

## Step 1: API 키 확인

다음 순서로 API 키를 로드합니다:

```bash
export PYTHONIOENCODING=utf-8
API_KEY="$LINEAR_API_KEY"
if [ -z "$API_KEY" ]; then
  API_KEY=$(python3 -c "
import json, os
p = os.path.expanduser('~/.config/linear/config.json')
if os.path.exists(p):
    print(json.load(open(p)).get('api_key', ''))
" 2>/dev/null)
fi
echo "현재 API_KEY: ${API_KEY:0:10}..."
```

비어 있으면 사용자에게 입력 요청:
> "Linear API 키를 입력하세요 (Linear 웹 → Settings → API → Personal API Keys):"

입력값을 `API_KEY`에 저장합니다.

## Step 2: 팀 목록 조회

```bash
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "import json; print(json.dumps({'query': '{ teams { nodes { id name key } } }'}))" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$RESPONSE"
```

응답에서 `{"errors":[...]}` 감지 시:
- 401 관련 메시지 포함: "API 키가 유효하지 않습니다. 다시 입력해주세요." → Step 1 반복
- 기타: 에러 내용 출력 후 중단

팀 목록을 번호와 함께 표시하고 선택을 요청합니다:
```
1) Adevi (ADE)
2) Backend (BKD)
팀을 선택하세요 (번호):
```
선택한 팀의 `id` → `TEAM_ID`, `key` → `TEAM_KEY`에 저장합니다.

## Step 3: 워크플로우 상태 조회 및 매핑

```bash
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
q = '''{ workflowStates(filter: { team: { id: { eq: \"''' + sys.argv[1] + '''\" } } }) { nodes { id name type } } }'''
print(json.dumps({'query': q}))
" "$TEAM_ID" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$RESPONSE"
```

상태 목록을 번호와 함께 표시합니다 (예):
```
1) Backlog   (backlog)
2) Todo      (unstarted)
3) In Progress (started)
4) In Review   (started)
5) Done        (completed)
6) Canceled    (cancelled)
7) Duplicate   (cancelled)
```

아래 4개 역할에 순서대로 번호를 입력받습니다:
```
"In Progress" 역할 번호:
"In Review" 역할 번호:
"Done" 역할 번호:
"Canceled" 역할 번호:
```

각 역할의 `id`와 `name`을 저장합니다.

## Step 4: Base Branch 자동 감지

```bash
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
echo "감지된 기준 브랜치: $BASE_BRANCH"
```

비어 있으면:
> "기준 브랜치를 입력하세요 (예: main, master):"

## Step 5: 설정 파일 저장

전역 설정 파일 저장:
```bash
mkdir -p ~/.config/linear
python3 -c "
import json, os, sys
p = os.path.expanduser('~/.config/linear/config.json')
config = json.load(open(p)) if os.path.exists(p) else {}
config['api_key'] = sys.argv[1]
with open(p, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('저장:', p)
" "$API_KEY"
```

프로젝트 설정 파일 저장:
```bash
mkdir -p .claude
python3 -c "
import json, sys
config = {
  'team_id': sys.argv[1],
  'team_key': sys.argv[2],
  'base_branch': sys.argv[3],
  'state_mapping': {
    'in_progress': {'id': sys.argv[4], 'name': sys.argv[5]},
    'in_review':   {'id': sys.argv[6], 'name': sys.argv[7]},
    'done':        {'id': sys.argv[8], 'name': sys.argv[9]},
    'canceled':    {'id': sys.argv[10], 'name': sys.argv[11]}
  }
}
with open('.claude/linear.json', 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('저장: .claude/linear.json')
" "$TEAM_ID" "$TEAM_KEY" "$BASE_BRANCH" \
  "$IN_PROGRESS_ID" "$IN_PROGRESS_NAME" \
  "$IN_REVIEW_ID" "$IN_REVIEW_NAME" \
  "$DONE_ID" "$DONE_NAME" \
  "$CANCELED_ID" "$CANCELED_NAME"
```

## 완료 메시지

다음 형식으로 출력합니다:
```
설정 완료!

팀: {TEAM_KEY}
기준 브랜치: {BASE_BRANCH}

사용 가능한 커맨드:
  /ls:list    — 이슈 목록
  /ls:start   — 작업 시작
```
