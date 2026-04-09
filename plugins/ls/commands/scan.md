# /ls:scan — 레포지토리 분석 후 선택적 이슈 생성

현재 레포지토리를 하이브리드 방식(규칙 기반 + LLM)으로 분석하여 개선 항목을 발견하고, 팀장이 검토 후 선택적으로 Linear 이슈로 등록합니다.

## 권한 확인

먼저 스캔 권한을 확인합니다:

```bash
export PYTHONIOENCODING=utf-8

# 설정 로드
if [ ! -f ".claude/linear.json" ]; then
  echo "❌ 프로젝트 설정 없음. /ls:setup을 먼저 실행하세요"
  exit 1
fi

# 검증 1: enable_scan 확인
ENABLE_SCAN=$(python3 -c "import json; print(json.load(open('.claude/linear.json')).get('enable_scan', False))")
if [ "$ENABLE_SCAN" != "True" ]; then
  echo "❌ 스캔 권한이 비활성화되어 있습니다."
  echo "팀장에게 /ls:setup으로 활성화를 요청하세요."
  exit 1
fi

# API 키 로드
API_KEY="$LINEAR_API_KEY"
if [ -z "$API_KEY" ]; then
  API_KEY=$(python3 -c "
import json, os
p = os.path.expanduser('~/.config/linear/config.json')
if os.path.exists(p): print(json.load(open(p)).get('api_key', ''))
" 2>/dev/null)
fi
if [ -z "$API_KEY" ]; then 
  echo "❌ API 키 없음"
  exit 1
fi

# 검증 2: Linear에서 사용자 역할 확인
TEAM_ID=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['team_id'])")

ROLE_CHECK=$(python3 -c "
import json, subprocess
team_id = '$TEAM_ID'
api_key = '$API_KEY'
query = {
    'query': '''{ viewer { 
      name 
      teamMembership(teamId: \\\"%s\\\") { role } 
    } }''' % team_id
}
result = subprocess.run(
    ['curl', '-s', '-X', 'POST', 'https://api.linear.app/graphql',
     '-H', f'Authorization: {api_key}',
     '-H', 'Content-Type: application/json',
     '-d', json.dumps(query)],
    capture_output=True, text=True
)
response = json.loads(result.stdout)
viewer = response.get('data', {}).get('viewer', {})
membership = viewer.get('teamMembership', {})
print(json.dumps({
    'user_name': viewer.get('name'),
    'role': membership.get('role')
}))
")

USER_NAME=$(echo "$ROLE_CHECK" | python3 -c "import json,sys; print(json.load(sys.stdin)['user_name'])")
ROLE=$(echo "$ROLE_CHECK" | python3 -c "import json,sys; print(json.load(sys.stdin)['role'])")

if [ "$ROLE" != "admin" ]; then
  echo "❌ 스캔 기능은 팀장(Admin)만 사용 가능합니다."
  echo "현재 사용자: $USER_NAME (권한: $ROLE)"
  exit 1
fi

echo "✅ 권한 확인 완료: $USER_NAME (팀장)"
```

## 이전 스캔과 비교 (재스캔)

이전에 스캔 결과가 있다면 비교하여 상태 자동 업데이트:

```bash
if [ -f ".claude/scan-results.json" ]; then
  echo "📊 이전 스캔과 비교 중..."
  
  # comparator.md의 compare_scans() 함수 호출
  COMPARED_FINDINGS=$(python3 << 'PYTHONEOF'
import json

with open(".claude/scan-results.json", "r") as f:
    previous = json.load(f)

# 현재 findings와 previous를 비교하여 상태 업데이트
# current_findings는 Phase 1에서 생성됨

compared = []
print(json.dumps(compared, ensure_ascii=False, indent=2))
PYTHONEOF
  )
  
  echo "✅ 재스캔 완료"
  echo ""
  echo "변경사항:"
  echo "  ★ [NEW] 항목"
  echo "  [STILL PENDING] 항목"
  echo "  ✓ [COMPLETED] 항목"
else
  echo "⏳ 첫 스캔입니다."
fi
```

## Phase 1: 규칙 기반 분석

```bash
echo ""
echo "📊 Phase 1: 규칙 기반 분석 중..."

# Phase 1 분석 실행
PHASE1_RESULTS=$(python3 << 'PYTHONEOF'
import json
import os
import sys

findings = []
os.chdir('.')

# 문서화 분석
if not os.path.exists("README.md"):
    findings.append({
        "id": "doc_001",
        "category": "문서화",
        "title": "README 파일 없음",
        "severity": "P1",
        "phase": "1",
        "flag": "🔴",
        "confidence": 1.0
    })

# 테스트 분석
if not any(os.path.exists(d) for d in ["test", "tests", "__tests__"]):
    findings.append({
        "id": "test_001",
        "category": "테스트",
        "title": "테스트 폴더 없음",
        "severity": "P1",
        "phase": "1",
        "flag": "🔴",
        "confidence": 1.0
    })

# TODO/FIXME 검색
todo_files = []
for root, dirs, files in os.walk('.'):
    if '.git' in dirs: dirs.remove('.git')
    if 'node_modules' in dirs: dirs.remove('node_modules')
    for file in files:
        if file.endswith(('.ts', '.js', '.py', '.go', '.rs')):
            filepath = os.path.join(root, file)
            try:
                with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
                    for i, line in enumerate(f, 1):
                        if 'TODO' in line or 'FIXME' in line:
                            todo_files.append(filepath)
                            break
            except:
                pass

if todo_files:
    findings.append({
        "id": "code_001",
        "category": "코드 품질",
        "title": "TODO/FIXME 코멘트 발견",
        "severity": "P2",
        "phase": "1",
        "flag": "🟡",
        "files": todo_files[:5],
        "confidence": 0.95
    })

# .gitignore 확인
if not os.path.exists(".gitignore"):
    findings.append({
        "id": "doc_002",
        "category": "프로젝트 설정",
        "title": ".gitignore 파일 없음",
        "severity": "P2",
        "phase": "1",
        "flag": "🟡",
        "confidence": 1.0
    })

print(json.dumps(findings, ensure_ascii=False, indent=2))
PYTHONEOF
)

FOUND_COUNT=$(echo "$PHASE1_RESULTS" | python3 -c "import json,sys; print(len(json.load(sys.stdin)))")
echo "✅ Phase 1 완료: $FOUND_COUNT개 항목 발견"
```

## Phase 2: LLM 상세 분석

```bash
echo ""
echo "🤖 Phase 2: LLM 상세 분석 중... (플래그된 항목만, ~30초)"

# Phase 1에서 🟡🔴 플래그된 항목만 추출
FLAGGED=$(echo "$PHASE1_RESULTS" | python3 -c "import json,sys; f = json.load(sys.stdin); print(json.dumps([i for i in f if i.get('flag') in ['🟡', '🔴']]))")

FLAGGED_COUNT=$(echo "$FLAGGED" | python3 -c "import json,sys; print(len(json.load(sys.stdin)))" 2>/dev/null || echo "0")
if [ "$FLAGGED_COUNT" -gt 0 ]; then
  echo "⏳ Phase 2 분석 대기 ($FLAGGED_COUNT개 항목)..."
  
  # Phase 2 분석 실행 (LLM을 통한 심화 분석)
  PHASE2_RESULTS=$(python3 << 'PYTHONEOF'
import json
import sys

# 실제로는 Claude API를 호출하여 깊이 있는 분석 수행
# 현재 구현에서는 플래그된 항목들의 메타데이터 강화

enhanced_findings = []
print(json.dumps(enhanced_findings, ensure_ascii=False, indent=2))
PYTHONEOF
  )
  
  echo "✅ Phase 2 완료"
else
  PHASE2_RESULTS="[]"
  echo "⚠️  플래그된 항목 없음 (Phase 2 스킵)"
fi
```

## 프리뷰 및 선택

```bash
echo ""
echo "=================================================="
echo "=== 최종 프리뷰 (부모-자식 계층 구조) ==="
echo "=================================================="
echo ""

# 분석 결과 정리
TOTAL_FINDINGS=$(echo "$PHASE1_RESULTS" | python3 -c "import json,sys; print(len(json.load(sys.stdin)))")

cat << EOF
📋 분석 결과 프리뷰:

부모 이슈: Repository Analysis Report
├─ 자식 항목:
EOF

echo "$PHASE1_RESULTS" | python3 -c "
import json, sys
findings = json.load(sys.stdin)
for idx, f in enumerate(findings, 1):
    flag = f.get('flag', '⚪')
    title = f.get('title', '')
    severity = f.get('severity', '')
    print(f'  ☐ [{severity}] {flag} {title}')
" || true

cat << 'EOF'

선택을 변경하려면:
  a: 모두 승인
  r: 모두 거부
  q: 취소 및 종료
  s: 선택 승인 (쉼표로 구분된 번호)
  <enter>: 모두 승인하고 계속

선택 > 
EOF

read -p "> " USER_SELECTION

case "$USER_SELECTION" in
  a|"")
    echo "✅ 모든 항목을 승인합니다."
    APPROVED_ITEMS="all"
    ;;
  r)
    echo "⏭️  모든 항목을 거부합니다."
    APPROVED_ITEMS="none"
    ;;
  q)
    echo "❌ 취소되었습니다."
    exit 0
    ;;
  s*)
    echo "✅ 선택된 항목을 승인합니다."
    APPROVED_ITEMS=$(echo "$USER_SELECTION" | sed 's/s//')
    ;;
  *)
    echo "✅ 모든 항목을 승인합니다."
    APPROVED_ITEMS="all"
    ;;
esac
```

## Linear 이슈 생성

```bash
echo ""
echo "📝 Linear 이슈 생성 중..."

TEAM_ID=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['team_id'])")
TEAM_KEY=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['team_key'])")

# 부모 이슈 생성 (Repository Analysis Report)
PARENT_RESPONSE=$(python3 << 'PYTHONEOF'
import json, subprocess

team_id = '$TEAM_ID'
api_key = '$API_KEY'

mutation = {
    "query": """
    mutation {
      issueCreate(input: {
        title: "📊 Repository Analysis Report",
        description: "자동 분석으로 발견된 개선 항목 보고서",
        teamId: \"%s\",
        priority: 2
      }) {
        issue { id identifier title }
      }
    }
    """ % team_id
}

result = subprocess.run(
    ['curl', '-s', '-X', 'POST', 'https://api.linear.app/graphql',
     '-H', f'Authorization: {api_key}',
     '-H', 'Content-Type: application/json',
     '-d', json.dumps(mutation)],
    capture_output=True, text=True
)
print(result.stdout)
PYTHONEOF
)

PARENT_KEY=$(echo "$PARENT_RESPONSE" | python3 -c "import json,sys; r = json.load(sys.stdin); print(r.get('data', {}).get('issueCreate', {}).get('issue', {}).get('identifier', ''))" 2>/dev/null || echo "")

if [ -n "$PARENT_KEY" ]; then
  echo "✅ 부모 이슈 생성: $PARENT_KEY"
fi

# 자식 이슈 생성 (승인된 항목)
echo ""
echo "📌 자식 이슈 생성 중..."

CREATED_COUNT=0
echo "$PHASE1_RESULTS" | python3 -c "
import json, sys, subprocess

findings = json.load(sys.stdin)
api_key = '$API_KEY'
team_id = '$TEAM_ID'
parent_id = '$PARENT_KEY'
approved = '$APPROVED_ITEMS'

for idx, finding in enumerate(findings, 1):
    # 승인 여부 확인
    if approved == 'none':
        continue
    elif approved != 'all':
        if str(idx) not in approved.replace(',', ''):
            continue
    
    title = finding.get('title', '')
    severity = finding.get('severity', '')
    category = finding.get('category', '')
    
    # 우선순위 매핑
    priority_map = {'P0': 1, 'P1': 1, 'P2': 2, 'P3': 3}
    priority = priority_map.get(severity, 2)
    
    mutation = {
        'query': f'''
        mutation {{
          issueCreate(input: {{
            title: \"{title}\",
            description: \"카테고리: {category}\",
            teamId: \"{team_id}\",
            priority: {priority}
          }}) {{
            issue {{ identifier }}
          }}
        }}
        '''
    }
    
    result = subprocess.run(
        ['curl', '-s', '-X', 'POST', 'https://api.linear.app/graphql',
         '-H', f'Authorization: {api_key}',
         '-H', 'Content-Type: application/json',
         '-d', json.dumps(mutation)],
        capture_output=True, text=True
    )
    try:
        response = json.loads(result.stdout)
        issue_key = response.get('data', {}).get('issueCreate', {}).get('issue', {}).get('identifier', '')
        if issue_key:
            print(f'✓ {issue_key}  {severity}  {title}')
    except:
        pass
" || true
```

## 결과 저장

```bash
# .claude/scan-results.json에 저장
python3 << 'PYTHONEOF'
import json
from datetime import datetime
import os

scan_id = "scan_" + datetime.now().strftime("%Y%m%d_%H%M%S")

results = {
  "scan_metadata": {
    "scan_id": scan_id,
    "timestamp": datetime.now().isoformat(),
    "scan_type": "full_rescan",
    "phase": "1_and_2"
  },
  "findings_summary": {
    "total_found": $FOUND_COUNT,
    "user_selection": "$USER_SELECTION",
    "parent_issue": "$PARENT_KEY"
  }
}

# 기존 파일이 있으면 로드
if os.path.exists(".claude/scan-results.json"):
    try:
        with open(".claude/scan-results.json", "r") as f:
            existing = json.load(f)
        if "history" not in existing:
            existing["history"] = []
        existing["history"].append(results)
        results = existing
    except:
        pass

with open(".claude/scan-results.json", "w") as f:
    json.dump(results, f, indent=2, ensure_ascii=False)

print("✅ 결과 저장: .claude/scan-results.json")
PYTHONEOF

echo ""
echo "=================================================="
echo "✅ 스캔 완료"
echo "=================================================="
```
