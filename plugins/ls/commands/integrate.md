# /ls:integrate [PARENT-KEY] — 서브이슈 통합 및 부모 PR 생성

You are executing `/ls:integrate [PARENT-KEY]`. Follow these steps in order. This merges sub-issue branches into the parent branch and creates a parent PR.

## Step 1: Load Configuration

Check for the Linear API key:

```bash
API_KEY="${LINEAR_API_KEY}"
if [ -z "$API_KEY" ]; then
  API_KEY=$(python3 -c "
import json, os
p = os.path.expanduser('~/.config/linear/config.json')
if os.path.exists(p):
    print(json.load(open(p)).get('api_key', ''))
" 2>/dev/null || echo "")
fi
echo "API_KEY_SET=${#API_KEY}"
```

If `API_KEY` is empty, tell the user: "Linear API 키가 설정되지 않았습니다. 먼저 /ls:setup을 실행하세요."

Also verify `gh` CLI is installed and authenticated.

Read `.claude/linear.json` and extract:
- `base_branch`
- `state_mapping.in_review.id` → `$IN_REVIEW_ID`

## Step 2: Resolve Parent Issue Key

If no `[PARENT-KEY]` argument was provided, try to extract it from the current Git branch:

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
PARENT_KEY=$(echo "$CURRENT_BRANCH" | grep -oE '^[a-z]+-[0-9]+' || echo "")
echo "PARENT_KEY=$PARENT_KEY"
```

If `PARENT_KEY` is still empty, ask the user:
> "부모 이슈 키를 입력하세요 (예: ADE-20):"

## Step 3: Fetch Parent Issue and Sub-Issues

Extract team and number from `$PARENT_KEY`. Run:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ issues(filter: {team: {key: {eq: \"TEAM\"}}, number: {eq: NUMBER}}) { nodes { id identifier title branchName state { name } children { nodes { identifier title branchName state { name type } } } comments(last: 5) { nodes { body user { name } } } } } }"}'
```

Parse the response and extract:
- `issues.nodes[0].id` → `$PARENT_UUID`
- `issues.nodes[0].title` → `$PARENT_TITLE`
- `issues.nodes[0].branchName` → `$PARENT_BRANCH_NAME`
- `issues.nodes[0].children.nodes[]` → sub-issues list

Display a table of sub-issues with their completion status:

```
서브이슈 현황:
  [x] ADE-30  완료  부모 이슈 준비
  [x] ADE-31  완료  로그 레벨별 필터
  [ ] ADE-32  대기  Sentry 연동
```

If any sub-issues are incomplete (not `completed` or `cancelled`), ask the user:
> "일부 서브이슈가 미완료 상태입니다. 완료된 것만 병합하시겠습니까? (Y/N):"

If they answer `N`, stop.

## Step 4: Identify Last Completed Sub-Issue Branch

From the sub-issues list, find the last one with completed/cancelled status. Since sub-issue branches are chained (each one includes prior changes), use its branch as the merge source:

- `$LAST_SUB_KEY` — last completed sub-issue key
- `$LAST_SUB_BRANCH` — corresponding branchName

Ask the user:
> "다음 브랜치를 부모 브랜치로 병합합니다: {LAST_SUB_BRANCH} → {PARENT_BRANCH_NAME}. 계속하시겠습니까? (Y/N):"

If they answer `N`, stop.

## Step 5: Checkout or Create Parent Branch

Run:

```bash
git show-ref --verify --quiet "refs/heads/{PARENT_BRANCH_NAME}"
```

- If exists, run: `git checkout {PARENT_BRANCH_NAME}`
- If not exists, run: `git checkout -b {PARENT_BRANCH_NAME} {BASE_BRANCH}`

Tell the user: "부모 브랜치 준비: {PARENT_BRANCH_NAME}"

## Step 6: Merge Sub-Issue Branch

Run:

```bash
git merge --no-ff {LAST_SUB_BRANCH} -m "Merge {LAST_SUB_KEY} sub-issues into {PARENT_KEY}"
```

If merge conflicts occur, tell the user:
> "병합 충돌이 발생했습니다. 수동으로 해결한 후 다시 시도하세요."

Stop and have the user resolve the conflict.

If merge succeeds, tell the user: "브랜치 병합 완료: {LAST_SUB_BRANCH} → {PARENT_BRANCH_NAME}"

## Step 7: Verify GitHub CLI and Push

Verify `gh` CLI is ready. Then run:

```bash
git push -u origin {PARENT_BRANCH_NAME}
```

If push fails, tell the user the error and stop.

## Step 8: Draft PR Body

Run:

```bash
git diff {BASE_BRANCH}...{PARENT_BRANCH_NAME} --stat
git diff {BASE_BRANCH}...{PARENT_BRANCH_NAME}
```

Compose a PR body with these sections:

- **Title**: `[{PARENT_KEY}] {PARENT_TITLE}`
- **Body**:
  - **개요** — summary from issue description
  - **완료된 서브이슈** — list completed sub-issues with `[x]`
  - **변경 사항** — bullets from diff
  - **Linear 이슈** — `https://linear.app/{PARENT_KEY}`

Show the draft to the user:
> "다음 내용으로 PR을 생성할까요? (Y/수정 요청):"

## Step 9: Create PR and Update Parent Issue

Once confirmed, run:

```bash
gh pr create \
  --title "[{PARENT_KEY}] {PARENT_TITLE}" \
  --body-file {body_file} \
  --base {BASE_BRANCH}
```

Capture the PR URL. Tell the user: "PR 생성됨: {PR_URL}"

Then update the parent issue state to "In Review":

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { issueUpdate(id: \"PARENT_UUID\", input: { stateId: \"IN_REVIEW_ID\" }) { issue { identifier state { name } } } }"}'
```

Tell the user: "{PARENT_KEY} 상태가 In Review로 변경되었습니다."
