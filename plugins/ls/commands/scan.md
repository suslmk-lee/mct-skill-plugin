# /ls:scan — 프로젝트 분석 후 이슈 일괄 등록

현재 프로젝트를 분석하여 발견한 작업 항목을 Linear 이슈로 분할 등록합니다.

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

## Step 2: 프로젝트 분석

다음 정보를 수집하여 이슈 후보를 도출합니다:

```bash
# Git 최근 커밋 (컨텍스트용)
git log --oneline -20 2>/dev/null

# 미완료 작업 힌트
grep -rn "TODO\|FIXME\|HACK\|XXX" --include="*.ts" --include="*.js" \
  --include="*.py" --include="*.go" --include="*.rs" \
  -l 2>/dev/null | head -20

# 변경된 파일 목록
git status --short 2>/dev/null

# README / 문서에서 알려진 이슈
cat README.md 2>/dev/null | head -100
```

수집한 정보와 사용자 요청을 종합하여 이슈 후보 목록을 작성합니다.

## Step 3: 이슈 후보 목록 표시 및 확인

다음 형식으로 이슈 후보를 출력하고 확인을 요청합니다:

```
발견된 작업 항목:

1. [제목] 짧은 설명 (우선순위: High/Medium/Low)
2. [제목] 짧은 설명 (우선순위: Medium)
3. [제목] 짧은 설명 (우선순위: Low)
...

위 항목을 Linear에 등록할까요?
- 전체 등록: y
- 선택 등록: 번호 입력 (예: 1,3,5)
- 취소: n
```

## Step 4: 이슈 일괄 등록

사용자가 확인한 항목에 대해 순서대로 등록합니다.

우선순위 매핑: `High=1, Medium=2, Low=3, Urgent=0`

```bash
TMPFILE=$(mktemp /tmp/linear-XXXXXX.json)
python3 -c "
import json, sys
title = sys.argv[1]
desc  = sys.argv[2]
team  = sys.argv[3]
prio  = int(sys.argv[4])
mutation = '''mutation {
  issueCreate(input: {
    title: \"''' + title.replace('\"','\\\\\"') + '''\",
    description: \"''' + desc.replace('\"','\\\\\"') + '''\",
    teamId: \"''' + team + '''\",
    priority: ''' + str(prio) + '''
  }) { issue { identifier title url } }
}'''
print(json.dumps({'query': mutation}, ensure_ascii=False))
" "$TITLE" "$DESCRIPTION" "$TEAM_ID" "$PRIORITY" > "$TMPFILE"
RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" -H "Content-Type: application/json" \
  --data-binary "@$TMPFILE")
rm -f "$TMPFILE"
echo "$RESPONSE"
```

각 이슈 등록 후 결과를 출력합니다:
```
✓ ADE-31  High    인증 토큰 만료 처리 누락
✓ ADE-32  Medium  TODO: 페이지네이션 미구현 (src/api/list.ts:42)
✓ ADE-33  Low     FIXME: 하드코딩된 타임아웃 값
```

## Step 5: 완료 요약

```
등록 완료: {N}개 이슈
팀: {TEAM_KEY}

작업을 시작하려면: /ls:start {ISSUE-KEY}
```
