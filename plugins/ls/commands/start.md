# /ls:start [ISSUE-KEY] — 작업 세션 시작

이슈 컨텍스트를 로드하고 Git 브랜치를 생성합니다.

## Step 1: 설정 로드

```bash
export PYTHONIOENCODING=utf-8
# API 키 로드
API_KEY="$LINEAR_API_KEY"
if [ -z "$API_KEY" ]; then
  API_KEY=$(python3 -c "
import json, os
p = os.path.expanduser('~/.config/linear/config.json')
if os.path.exists(p): print(json.load(open(p)).get('api_key', ''))
" 2>/dev/null)
fi
if [ -z "$API_KEY" ]; then echo "API 키 없음. /ls:setup을 먼저 실행하세요"; exit 1; fi

# 프로젝트 설정 로드
if [ ! -f ".claude/linear.json" ]; then echo "프로젝트 설정 없음. /ls:setup을 먼저 실행하세요"; exit 1; fi
TEAM_KEY=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['team_key'])")
BASE_BRANCH=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['base_branch'])")
IN_PROGRESS_ID=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['state_mapping']['in_progress']['id'])")
```

## Step 2: 이슈 선택

ISSUE-KEY 인자가 없으면 나에게 할당된 이슈 목록을 표시합니다:

```bash
# 현재 사용자 ID 조회
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "import json; print(json.dumps({'query': '{ viewer { id name } }'}))" > "$TMPFILE"
VIEWER_RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
VIEWER_ID=$(echo "$VIEWER_RESPONSE" | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['viewer']['id'])")

# 할당된 이슈 목록 조회
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
q = '''{ issues(first: 20, filter: {
  assignee: { id: { eq: \"''' + sys.argv[1] + '''\" } },
  state: { type: { in: [\"backlog\", \"unstarted\"] } }
}) { nodes { identifier title branchName state { name } } } }'''
print(json.dumps({'query': q}))
" "$VIEWER_ID" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$RESPONSE"
```

이슈 목록을 번호와 함께 표시하고 선택을 요청합니다:
```
1) ADE-24  Backlog  로그 레벨 분리
2) ADE-25  Todo     워크플로우 히스토리 UI
시작할 이슈 번호:
```

## Step 3: 이슈 UUID + 상세 조회

선택한 이슈 키(예: ADE-24)에서 UUID를 조회하고 상세 정보를 가져옵니다:

```bash
ISSUE_KEY_LOWER=$(echo "$ISSUE_KEY" | tr '[:upper:]' '[:lower:]')
ISSUE_TEAM=$(echo "$ISSUE_KEY_LOWER" | cut -d'-' -f1)
ISSUE_NUM=$(echo "$ISSUE_KEY_LOWER" | cut -d'-' -f2)

TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
team, num = sys.argv[1], int(sys.argv[2])
q = '''{ issues(filter: {
  team: { key: { eq: \"''' + team + '''\" } },
  number: { eq: ''' + str(num) + ''' }
}) { nodes {
  id identifier title description branchName
  parent { id identifier branchName }
  state { id name }
  comments(last: 5) { nodes { body user { name } } }
} } }'''
print(json.dumps({'query': q}))
" "$ISSUE_TEAM" "$ISSUE_NUM" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$RESPONSE"
```

응답에서 추출:
- `ISSUE_UUID`: `data.issues.nodes[0].id`
- `ISSUE_TITLE`: `data.issues.nodes[0].title`
- `ISSUE_DESC`: `data.issues.nodes[0].description`
- `BRANCH_NAME`: `data.issues.nodes[0].branchName`
- `PARENT_KEY`: `data.issues.nodes[0].parent.identifier` (없으면 빈 문자열)
- `PARENT_BRANCH`: `data.issues.nodes[0].parent.branchName` (없으면 빈 문자열)
- 수락기준: `ISSUE_DESC`에서 `- [ ]` 패턴 줄 추출
- 댓글: `data.issues.nodes[0].comments.nodes[]`

이슈를 찾지 못하면: "이슈를 찾을 수 없습니다: {ISSUE_KEY}" 출력 후 중단.

## Step 4: Dirty Working Tree 확인

```bash
git status --porcelain
```

출력이 비어 있지 않으면:
> "커밋하지 않은 변경사항이 있습니다. 계속 진행하시겠습니까? (y/n):"

n이면 중단합니다.

## Step 5: 브랜치 생성 또는 전환

```bash
# 서브이슈이고 브랜치가 아직 없을 때만 묻는다
if [ -n "$PARENT_KEY" ] && ! git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"; then
  CURRENT_BRANCH=$(git branch --show-current 2>/dev/null)
  echo "이 이슈는 [$PARENT_KEY]의 서브이슈입니다. 분기 기준 브랜치를 선택하세요:"
  echo "  1) 현재 브랜치: $CURRENT_BRANCH"
  echo "  2) 부모 이슈 브랜치: $PARENT_BRANCH"
  echo "  3) 기본 브랜치: $BASE_BRANCH"
  # 사용자 입력 대기 → BRANCH_BASE 설정
  # 입력이 1 또는 엔터 → BRANCH_BASE="$CURRENT_BRANCH"
  # 입력이 2 → BRANCH_BASE="$PARENT_BRANCH"
  # 입력이 3 → BRANCH_BASE="$BASE_BRANCH"
else
  BRANCH_BASE="$BASE_BRANCH"
fi

if git show-ref --verify --quiet "refs/heads/$BRANCH_NAME"; then
  git checkout "$BRANCH_NAME"
  echo "기존 브랜치로 전환: $BRANCH_NAME"
else
  git checkout -b "$BRANCH_NAME" "$BRANCH_BASE"
  echo "새 브랜치 생성: $BRANCH_NAME"
fi
```

## Step 6: 이슈 상태 → In Progress

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
" "$ISSUE_UUID" "$IN_PROGRESS_ID" > "$TMPFILE"
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE" > /dev/null
rm -f "$TMPFILE"
```

## Step 7: 세션 컨텍스트 블록 출력

수락기준은 description에서 `- [ ]` 패턴을 추출합니다. 없으면 이 항목을 생략합니다.
댓글은 `@이름 "내용"` 형식으로 출력합니다.

다음 형식으로 출력합니다:

```
## 현재 작업 세션
이슈: {ISSUE_KEY} — {ISSUE_TITLE}
부모 이슈: {PARENT_KEY}                 ← parent가 있을 때만
설명: {ISSUE_DESC 첫 두 줄}
수락기준:                          ← description에 - [ ] 항목이 있을 때만
  [ ] ...
  [ ] ...
최근 댓글: @{name} "{body}"        ← 댓글이 있을 때만
브랜치: {BRANCH_NAME}
```

이 블록을 이후 대화에서 계속 참조합니다.
