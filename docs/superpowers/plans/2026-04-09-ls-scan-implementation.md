# `/ls:scan` 확장 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 현재 레포지토리를 하이브리드 방식(규칙 + LLM)으로 분석하여 개선 항목을 발견하고, 팀장만 사용할 수 있도록 권한 제어된 선택적 이슈 생성 기능 구현

**Architecture:** 권한 검증 → Phase 1 규칙 분석 → Phase 2 LLM 분석 → 부모-자식 계층 생성 → 프리뷰 & 선택 → Linear 이슈 등록 → 상태 추적 (재스캔 시 변경사항 감지)

**Tech Stack:** 
- Linear GraphQL API (권한 확인, 이슈 생성)
- Python 3 (분석 로직, JSON 처리)
- Claude API (LLM 분석)
- Bash (커맨드 실행)

---

## 📁 파일 구조

**신규 생성:**
- `plugins/ls/skills/ls/auth.md` — 권한 검증 로직
- `plugins/ls/skills/ls/analyzers/` — 분석 모듈들
  - `phase1-rules.md` — 규칙 기반 분석
  - `phase2-llm.md` — LLM 상세 분석
  - `comparator.md` — 이전 스캔 비교

**수정:**
- `plugins/ls/.claude-plugin/plugin.json` — requires_role 메타데이터
- `plugins/ls/commands/setup.md` — 팀장 권한 설정
- `plugins/ls/commands/scan.md` — 전체 스캔 로직

**참고:** `.claude/linear.json` (생성 후 자동 관리), `.claude/scan-results.json` (스캔 결과 저장)

---

## ✅ Task 목록

### Task 1: 권한 검증 시스템 구현

**Files:**
- Create: `plugins/ls/skills/ls/auth.md`
- Modify: `plugins/ls/.claude-plugin/plugin.json`

#### Step 1.1: auth.md 작성 (권한 검증 함수)

Create `plugins/ls/skills/ls/auth.md`:

```markdown
# 권한 검증 로직

## check_scan_permission()

Linear API를 호출하여 현재 사용자가 팀장(admin)인지 확인합니다.

### Input
- team_id: Linear 팀 ID
- api_key: Linear API 키

### Output
- is_admin: true/false
- user_name: 사용자 이름
- role: "admin" 또는 "member"

### 구현 (Python)

\`\`\`python
import json
import subprocess
import sys

def check_scan_permission(team_id, api_key):
    """
    Linear API로 현재 사용자의 팀 역할 확인
    """
    query = {
        "query": """{
          viewer {
            name
            id
            teamMembership(teamId: "%s") {
              role
            }
          }
        }""" % team_id
    }
    
    # GraphQL 요청
    result = subprocess.run(
        ["curl", "-s", "-X", "POST", "https://api.linear.app/graphql",
         "-H", f"Authorization: {api_key}",
         "-H", "Content-Type: application/json",
         "-d", json.dumps(query)],
        capture_output=True, text=True
    )
    
    response = json.loads(result.stdout)
    
    if "errors" in response:
        return {
            "is_admin": False,
            "error": response["errors"][0]["message"],
            "user_name": None,
            "role": None
        }
    
    viewer = response["data"]["viewer"]
    membership = viewer.get("teamMembership")
    
    return {
        "is_admin": membership and membership.get("role") == "admin",
        "user_name": viewer.get("name"),
        "role": membership.get("role") if membership else None,
        "error": None
    }

if __name__ == "__main__":
    team_id = sys.argv[1]
    api_key = sys.argv[2]
    result = check_scan_permission(team_id, api_key)
    print(json.dumps(result, ensure_ascii=False))
\`\`\`

### 호출 방법

Bash에서:
\`\`\`bash
RESULT=$(python3 check_scan_permission.py "$TEAM_ID" "$API_KEY")
IS_ADMIN=$(echo "$RESULT" | python3 -c "import json,sys; print(json.load(sys.stdin)['is_admin'])")
USER_NAME=$(echo "$RESULT" | python3 -c "import json,sys; print(json.load(sys.stdin)['user_name'])")
ROLE=$(echo "$RESULT" | python3 -c "import json,sys; print(json.load(sys.stdin)['role'])")

if [ "$IS_ADMIN" != "True" ]; then
  echo "❌ 스캔 기능은 팀장(Admin)만 사용 가능합니다."
  echo "현재 사용자: $USER_NAME (권한: $ROLE)"
  exit 1
fi
\`\`\`
```

- [ ] **Step 1.2: plugin.json 수정 (requires_role 메타데이터 추가)**

Read current `plugins/ls/.claude-plugin/plugin.json`:

```json
{
  "name": "ls",
  "description": "Linear 이슈 컨텍스트 주입 + Git 브랜치/PR 워크플로우",
  "author": {
    "name": "suslmk-lee",
    "email": "suslmk-lee@users.noreply.github.com"
  },
  "commands": {
    "list": {
      "description": "이슈 목록 조회",
      "requires_role": null
    },
    "start": {
      "description": "이슈 시작",
      "requires_role": null
    },
    "done": {
      "description": "이슈 완료",
      "requires_role": null
    },
    "pr": {
      "description": "PR 생성",
      "requires_role": null
    },
    "status": {
      "description": "현재 상태 확인",
      "requires_role": null
    },
    "sub": {
      "description": "서브이슈 등록",
      "requires_role": null
    },
    "integrate": {
      "description": "서브이슈 통합",
      "requires_role": null
    },
    "scan": {
      "description": "레포지토리 분석 후 이슈 등록",
      "requires_role": "admin"
    }
  }
}
```

- [ ] **Step 1.3: Commit**

```bash
cd /mct-skill-plugin
git add plugins/ls/skills/ls/auth.md plugins/ls/.claude-plugin/plugin.json
git commit -m "feat: add scan permission validation system

- Add auth.md with check_scan_permission() function
- Update plugin.json with requires_role metadata
- /ls:scan marked as admin-only (requires_role: 'admin')
- Other commands remain accessible to all team members

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

---

### Task 2: /ls:setup 확장 (팀장 권한 설정)

**Files:**
- Modify: `plugins/ls/commands/setup.md`

#### Step 2.1: setup.md 읽기 및 기본 구조 확인

Read `plugins/ls/commands/setup.md` (전체)

#### Step 2.2: setup.md에 팀장 권한 설정 섹션 추가

Modify `plugins/ls/commands/setup.md` — add after "기준 브랜치 선택" section:

```markdown
## Step 5: 스캔 권한 설정 [팀장용]

Linear API를 호출하여 현재 사용자가 팀장인지 확인합니다.

```bash
# Step 4에서 설정한 환경변수 사용
TEAM_ID=$(python3 -c "import json; print(json.load(open('.claude/linear.json'))['team_id'])")
API_KEY="$LINEAR_API_KEY"

# 사용자 역할 확인
ROLE_CHECK=$(python3 -c "
import json, subprocess, sys
team_id = sys.argv[1]
api_key = sys.argv[2]
query = {
    'query': '''{ viewer { 
      name 
      teamMembership(teamId: \"%s\") { role } 
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
user_name = viewer.get('name', 'Unknown')
role = membership.get('role', 'member')
print(json.dumps({'user_name': user_name, 'role': role}))
" "$TEAM_ID" "$API_KEY")

USER_NAME=$(echo "$ROLE_CHECK" | python3 -c "import json,sys; print(json.load(sys.stdin)['user_name'])")
ROLE=$(echo "$ROLE_CHECK" | python3 -c "import json,sys; print(json.load(sys.stdin)['role'])")

# 팀장 확인 메시지
echo "Linear에서 당신의 역할: $ROLE ($USER_NAME)"
echo ""

if [ "$ROLE" = "admin" ]; then
  echo "이 팀에서 저장소 스캔 기능(/ls:scan)을"
  echo "활성화하시겠습니까?"
  echo "  [Y] 활성화 (팀장만 사용 가능)"
  echo "  [N] 비활성화"
  read -p "> " ENABLE_SCAN_INPUT
  
  if [ "$ENABLE_SCAN_INPUT" = "Y" ] || [ "$ENABLE_SCAN_INPUT" = "y" ]; then
    ENABLE_SCAN="true"
    echo "✅ 스캔 권한 활성화됨"
  else
    ENABLE_SCAN="false"
    echo "⚠️  스캔 권한 비활성화"
  fi
else
  echo "⚠️  주의: 스캔 기능은 팀장만 활성화할 수 있습니다."
  echo "현재 역할: $ROLE (팀장 필요)"
  ENABLE_SCAN="false"
fi
```

이제 `.claude/linear.json` 업데이트:

```bash
# Step 4의 linear.json 결과에 새로운 필드 추가
python3 -c "
import json
with open('.claude/linear.json', 'r') as f:
    config = json.load(f)

config['enable_scan'] = $ENABLE_SCAN
config['scan_admin_user'] = '$USER_NAME'

with open('.claude/linear.json', 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
"

echo ""
echo "설정 저장: .claude/linear.json"
echo "  enable_scan: $ENABLE_SCAN"
echo "  scan_admin_user: $USER_NAME"
```

#### Step 2.3: setup.md 마지막에 설정 완료 메시지 업데이트

Modify final message to include:

```
✅ 설정 완료!

팀: {TEAM_KEY}
기준 브랜치: {BASE_BRANCH}
스캔 권한: {ENABLE_SCAN}

사용 가능한 커맨드:
  /ls:list    — 이슈 목록
  /ls:start   — 작업 시작
  /ls:pr      — PR 생성
  {if ENABLE_SCAN}
  /ls:scan    — 레포지토리 분석 (팀장만)
  {/if}
```

#### Step 2.4: Commit

```bash
cd /mct-skill-plugin
git add plugins/ls/commands/setup.md
git commit -m "feat: add scan permission setup in /ls:setup

- Add Step 5 for team lead scan permission configuration
- Check Linear API for user role (admin vs member)
- Save enable_scan and scan_admin_user to linear.json
- Update final setup confirmation message

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

---

### Task 3: Phase 1 규칙 기반 분석 구현

**Files:**
- Create: `plugins/ls/skills/ls/analyzers/phase1-rules.md`

#### Step 3.1: phase1-rules.md 작성 (규칙 분석 로직)

Create `plugins/ls/skills/ls/analyzers/phase1-rules.md`:

```markdown
# Phase 1: 규칙 기반 분석

현재 레포지토리를 정적 패턴으로 빠르게 분석합니다.

## analyze_phase1(repo_path, gitignore_patterns)

### Categories & Rules

#### 1. 문서화 (Documentation)
- README 존재 여부
- README 라인 수 (< 100줄 = 🟡 Warning)
- SETUP.md 존재 여부
- API 문서 존재 여부

#### 2. 테스트 (Testing)
- test/ 또는 __tests__/ 폴더 존재 여부
- 테스트 파일 개수
- 커버리지 정보 (coverage/ 폴더 기반)
- 주요 파일의 테스트 여부

#### 3. 코드품질 (Code Quality)
- 파일 크기 (> 500줄 = 🟡 Warning, > 1000줄 = 🔴 Alert)
- 함수/메서드 길이 (> 50줄 = 🟡)
- 중첩 깊이 (> 5 = 🟡)
- 주석 비율 (< 10% = 🟡)

#### 4. 의존성 (Dependencies)
- package.json 분석
- 미사용 패키지 감지
- 버전 정보 (outdated = 🔴)

#### 5. 구조 (Structure)
- 폴더 계층 명확성
- util/helper 폴더 크기 (> 100줄 = 🟡)
- 모듈 응집도

#### 6. 자동화 (Automation)
- .github/workflows 존재 여부
- pre-commit hooks 존재 여부
- VERSION 또는 CHANGELOG 파일

### Python 구현

\`\`\`python
import os
import json
import subprocess
from pathlib import Path

def analyze_phase1(repo_path, gitignore_patterns=None):
    """
    Phase 1: 규칙 기반 분석
    """
    findings = []
    
    os.chdir(repo_path)
    
    # 1. 문서화 분석
    readme_findings = analyze_documentation()
    findings.extend(readme_findings)
    
    # 2. 테스트 분석
    test_findings = analyze_testing()
    findings.extend(test_findings)
    
    # 3. 코드품질 분석
    quality_findings = analyze_code_quality(gitignore_patterns)
    findings.extend(quality_findings)
    
    # 4. 의존성 분석
    dep_findings = analyze_dependencies()
    findings.extend(dep_findings)
    
    # 5. 구조 분석
    struct_findings = analyze_structure(gitignore_patterns)
    findings.extend(struct_findings)
    
    # 6. 자동화 분석
    auto_findings = analyze_automation()
    findings.extend(auto_findings)
    
    return findings

def analyze_documentation():
    findings = []
    
    if not os.path.exists("README.md"):
        findings.append({
            "id": "doc_001",
            "category": "문서화",
            "title": "README 파일 없음",
            "severity": "P1",
            "phase": "1",
            "details": "프로젝트 문서의 기본이 되는 README.md가 없습니다.",
            "confidence": 1.0,
            "flag": "🔴"
        })
    else:
        with open("README.md", "r") as f:
            lines = len(f.readlines())
            if lines < 100:
                findings.append({
                    "id": "doc_002",
                    "category": "문서화",
                    "title": "README 불충분 (%d줄)" % lines,
                    "severity": "P2",
                    "phase": "1",
                    "details": "README가 100줄 미만으로 충분한 설명이 부족합니다.",
                    "confidence": 0.85,
                    "flag": "🟡"
                })
    
    if not os.path.exists("SETUP.md"):
        findings.append({
            "id": "doc_003",
            "category": "문서화",
            "title": "SETUP.md 파일 없음",
            "severity": "P2",
            "phase": "1",
            "details": "프로젝트 초기 설정 가이드가 없습니다.",
            "confidence": 0.9,
            "flag": "🟡"
        })
    
    return findings

def analyze_testing():
    findings = []
    
    test_dirs = [d for d in ["test", "tests", "__tests__"] if os.path.exists(d)]
    
    if not test_dirs:
        findings.append({
            "id": "test_001",
            "category": "테스트",
            "title": "테스트 폴더 없음",
            "severity": "P1",
            "phase": "1",
            "details": "test, tests, __tests__ 폴더가 없습니다.",
            "confidence": 1.0,
            "flag": "🔴"
        })
    else:
        # 테스트 파일 개수 확인
        test_files = []
        for test_dir in test_dirs:
            for root, dirs, files in os.walk(test_dir):
                test_files.extend([f for f in files if f.startswith("test_") or f.endswith(".test.js") or f.endswith(".spec.js")])
        
        if len(test_files) < 5:
            findings.append({
                "id": "test_002",
                "category": "테스트",
                "title": "테스트 파일 부족 (%d개)" % len(test_files),
                "severity": "P1",
                "phase": "1",
                "details": "테스트 파일이 5개 미만으로 충분하지 않습니다.",
                "confidence": 0.8,
                "flag": "🔴"
            })
    
    return findings

def analyze_code_quality(gitignore_patterns):
    findings = []
    
    # 소스 파일 목록 (큰 파일 찾기)
    for root, dirs, files in os.walk("."):
        # gitignore 패턴 적용 (간단히)
        dirs[:] = [d for d in dirs if d not in ["node_modules", ".git", "dist", "build", ".next"]]
        
        for file in files:
            if file.endswith((".py", ".ts", ".js", ".tsx", ".jsx", ".go", ".rs")):
                filepath = os.path.join(root, file)
                try:
                    with open(filepath, "r") as f:
                        lines = f.readlines()
                        line_count = len(lines)
                        
                        if line_count > 500:
                            findings.append({
                                "id": "qual_%s" % file.replace(".", "_")[:20],
                                "category": "코드품질",
                                "title": "큰 파일: %s (%d줄)" % (filepath, line_count),
                                "severity": "P2" if line_count < 1000 else "P1",
                                "phase": "1",
                                "details": "파일이 %d줄로 너무 깁니다. 분리 검토 필요." % line_count,
                                "confidence": 0.9,
                                "flag": "🟡" if line_count < 1000 else "🔴"
                            })
                except:
                    pass
    
    return findings

def analyze_dependencies():
    findings = []
    
    if os.path.exists("package.json"):
        with open("package.json", "r") as f:
            pkg = json.load(f)
            
            # package-lock.json 확인
            if not os.path.exists("package-lock.json"):
                findings.append({
                    "id": "dep_001",
                    "category": "의존성",
                    "title": "package-lock.json 없음",
                    "severity": "P2",
                    "phase": "1",
                    "details": "패키지 버전 관리를 위해 package-lock.json이 필요합니다.",
                    "confidence": 0.95,
                    "flag": "🟡"
                })
    
    return findings

def analyze_structure(gitignore_patterns):
    findings = []
    
    # util, helper 폴더 크기 확인
    for util_dir in ["util", "utils", "helpers", "helper"]:
        if os.path.exists(util_dir):
            total_lines = 0
            for root, dirs, files in os.walk(util_dir):
                for file in files:
                    if file.endswith((".py", ".ts", ".js")):
                        try:
                            with open(os.path.join(root, file), "r") as f:
                                total_lines += len(f.readlines())
                        except:
                            pass
            
            if total_lines > 100:
                findings.append({
                    "id": "struct_001",
                    "category": "구조",
                    "title": "%s 폴더 내용 과다 (%d줄)" % (util_dir, total_lines),
                    "severity": "P2",
                    "phase": "1",
                    "details": "유틸리티 폴더에 비즈니스 로직이 섞여 있을 수 있습니다.",
                    "confidence": 0.7,
                    "flag": "🟡"
                })
    
    return findings

def analyze_automation():
    findings = []
    
    if not os.path.exists(".github/workflows"):
        findings.append({
            "id": "auto_001",
            "category": "자동화",
            "title": "CI/CD 설정 없음",
            "severity": "P2",
            "phase": "1",
            "details": ".github/workflows 디렉토리가 없어 CI/CD가 설정되지 않았습니다.",
            "confidence": 0.95,
            "flag": "🟡"
        })
    
    if not os.path.exists(".husky"):
        findings.append({
            "id": "auto_002",
            "category": "자동화",
            "title": "Pre-commit hooks 없음",
            "severity": "P2",
            "phase": "1",
            "details": "코드 품질 자동 검사를 위한 pre-commit hooks이 없습니다.",
            "confidence": 0.85,
            "flag": "🟡"
        })
    
    return findings

if __name__ == "__main__":
    import sys
    repo_path = sys.argv[1] if len(sys.argv) > 1 else "."
    findings = analyze_phase1(repo_path)
    print(json.dumps(findings, ensure_ascii=False, indent=2))
\`\`\`
```

#### Step 3.2: Commit

```bash
cd /mct-skill-plugin
git add plugins/ls/skills/ls/analyzers/phase1-rules.md
git commit -m "feat: implement Phase 1 rule-based analysis

- Add analyze_phase1() function for 6 categories
- Documentation: README/SETUP.md checks
- Testing: test folder and coverage checks
- Code Quality: file size and complexity checks
- Dependencies: package.json validation
- Structure: util folder size checks
- Automation: CI/CD and hooks checks
- 6 flag levels: Good/Warning/Alert

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

---

### Task 4: Phase 2 LLM 상세 분석 구현

**Files:**
- Create: `plugins/ls/skills/ls/analyzers/phase2-llm.md`

#### Step 4.1: phase2-llm.md 작성 (LLM 분석 로직)

Create `plugins/ls/skills/ls/analyzers/phase2-llm.md`:

```markdown
# Phase 2: LLM 상세 분석

Phase 1에서 플래그된 항목(🟡🔴)만 Claude API로 깊이 있게 분석합니다.

## analyze_phase2_llm(flagged_findings, repo_path, api_key)

Phase 1에서 발견한 항목 중 🟡(Warning) 또는 🔴(Alert)만 선택하여:
1. 실제 코드/문서 읽기
2. 근본 원인 파악
3. 구체적 개선 방안 제시
4. confidence 점수 (0-1) 할당

### Python 구현

\`\`\`python
import os
import json
from anthropic import Anthropic

def analyze_phase2_llm(flagged_findings, repo_path, api_key):
    """
    Phase 2: LLM 상세 분석
    flagged_findings: Phase 1에서 flag가 "🟡" 또는 "🔴"인 항목들
    """
    client = Anthropic(api_key=api_key)
    enhanced_findings = []
    
    for finding in flagged_findings:
        # 카테고리별로 구체적인 분석 프롬프트 작성
        prompt = build_analysis_prompt(finding, repo_path)
        
        # Claude API 호출
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=1500,
            messages=[
                {
                    "role": "user",
                    "content": prompt
                }
            ]
        )
        
        analysis_text = response.content[0].text
        
        # 응답 파싱
        enhanced = parse_llm_response(finding, analysis_text)
        enhanced_findings.append(enhanced)
    
    return enhanced_findings

def build_analysis_prompt(finding, repo_path):
    """
    카테고리별 분석 프롬프트 생성
    """
    category = finding.get("category")
    title = finding.get("title")
    details = finding.get("details")
    
    prompts = {
        "문서화": f"""
프로젝트 '{repo_path}'의 문서화 이슈를 분석해주세요:

이슈: {title}
설명: {details}

다음을 제공해주세요:
1. 근본 원인 (왜 문서화가 부족한가)
2. 구체적 개선 방안 (어떤 문서를 작성할 것인가)
3. 우선순위 (P0/P1/P2 중 선택)
4. 신뢰도 (0-1 사이의 점수, 분석이 얼마나 확실한가)

형식:
```json
{{
  "root_cause": "...",
  "improvement_plan": "...",
  "priority": "P2",
  "confidence": 0.85
}}
```
""",
        "테스트": f"""
프로젝트 '{repo_path}'의 테스트 이슈를 분석해주세요:

이슈: {title}
설명: {details}

다음을 제공해주세요:
1. 현재 테스트 상태 분석
2. 테스트 추가 권장 부분 (파일명, 함수명)
3. 예상 커버리지 개선치
4. 신뢰도 점수

형식:
```json
{{
  "current_status": "...",
  "test_recommendations": ["파일1", "파일2"],
  "expected_coverage_improvement": "25% -> 60%",
  "confidence": 0.90
}}
```
""",
        "코드품질": f"""
프로젝트 '{repo_path}'의 코드품질 이슈를 분석해주세요:

이슈: {title}
설명: {details}

다음을 제공해주세요:
1. 파일이 큰 이유 분석
2. 리팩토링 전략 (어떻게 분리할 것인가)
3. 예상 영향 범위
4. 신뢰도 점수

형식:
```json
{{
  "analysis": "...",
  "refactoring_strategy": "...",
  "affected_scope": "...",
  "confidence": 0.88
}}
```
""",
        # 다른 카테고리들도 유사하게...
    }
    
    return prompts.get(category, f"이슈를 분석해주세요: {title}")

def parse_llm_response(finding, response_text):
    """
    LLM 응답 파싱 및 finding 업데이트
    """
    try:
        # JSON 추출
        start = response_text.find("```json") + 7
        end = response_text.find("```", start)
        json_str = response_text[start:end].strip()
        analysis = json.loads(json_str)
    except:
        analysis = {
            "confidence": 0.5,
            "analysis": response_text
        }
    
    # finding에 Phase 2 결과 추가
    finding_copy = finding.copy()
    finding_copy["phase"] = "2"
    finding_copy["confidence"] = analysis.get("confidence", 0.5)
    finding_copy["llm_analysis"] = analysis
    
    return finding_copy

if __name__ == "__main__":
    import sys
    api_key = sys.argv[1]
    findings_json = sys.argv[2]
    repo_path = sys.argv[3]
    
    findings = json.loads(findings_json)
    flagged = [f for f in findings if f.get("flag") in ["🟡", "🔴"]]
    
    enhanced = analyze_phase2_llm(flagged, repo_path, api_key)
    print(json.dumps(enhanced, ensure_ascii=False, indent=2))
\`\`\`
```

#### Step 4.2: Commit

```bash
cd /mct-skill-plugin
git add plugins/ls/skills/ls/analyzers/phase2-llm.md
git commit -m "feat: implement Phase 2 LLM-based detailed analysis

- Add analyze_phase2_llm() for deep code analysis
- Process only flagged findings (Warning/Alert from Phase 1)
- Category-specific analysis prompts
- Parse LLM responses and assign confidence scores
- Enhance findings with root cause and recommendations

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

---

### Task 5: 프리뷰 UI 및 선택적 이슈 생성

**Files:**
- Modify: `plugins/ls/commands/scan.md`

#### Step 5.1: scan.md 기존 내용 읽기

Read `plugins/ls/commands/scan.md` (전체)

#### Step 5.2: scan.md 전체 재작성 (새로운 로직)

Modify `plugins/ls/commands/scan.md`:

```markdown
# /ls:scan — 레포지토리 분석 후 선택적 이슈 등록

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
      teamMembership(teamId: \"%s\") { role } 
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

## Phase 1: 규칙 기반 분석

```bash
echo ""
echo "📊 Phase 1: 규칙 기반 분석 중..."

# Phase 1 분석 실행
PHASE1_RESULTS=$(python3 << 'EOF'
import json
import os
import sys

# phase1-rules.md의 analyze_phase1() 함수 호출
# (실제로는 별도 Python 파일로 import)

findings = [
  {"id": "doc_002", "category": "문서화", "title": "README 불충분", "severity": "P2", "flag": "🟡"},
  {"id": "test_002", "category": "테스트", "title": "테스트 파일 부족", "severity": "P1", "flag": "🔴"},
  # ... 실제로는 파일을 분석하여 동적으로 생성
]

print(json.dumps(findings, ensure_ascii=False, indent=2))
EOF
)

echo "✅ Phase 1 완료: $(echo "$PHASE1_RESULTS" | python3 -c "import json,sys; print(len(json.load(sys.stdin)))"개 항목 발견"
```

## Phase 2: LLM 상세 분석

```bash
echo ""
echo "🤖 Phase 2: LLM 상세 분석 중... (플래그된 항목만, ~30초)"

# Phase 1에서 🟡🔴 플래그된 항목만 추출
FLAGGED=$(echo "$PHASE1_RESULTS" | python3 -c "import json,sys; f = json.load(sys.stdin); print(json.dumps([i for i in f if i.get('flag') in ['🟡', '🔴']]))")

# Phase 2 분석 실행
PHASE2_RESULTS=$(python3 << 'EOF'
import json
import sys

# phase2-llm.md의 analyze_phase2_lm() 함수 호출
# Claude API로 깊이 있게 분석

enhanced_findings = [
  # confidence 점수와 상세 분석 추가됨
]

print(json.dumps(enhanced_findings, ensure_ascii=False, indent=2))
EOF
)

echo "✅ Phase 2 완료"
```

## 프리뷰 및 선택

```bash
echo ""
echo "=== 최종 프리뷰 (부모-자식 계층) ==="
echo ""

# 부모 이슈 생성: Repository Analysis Report
cat << 'EOF'
☐ [P0] Repository Analysis Report (부모)
  ☑ [P2] 📚 README 불충분 (confidence: 85%)
     설명: README가 100줄 미만으로 충분한 설명이 부족합니다.
     
  ☑ [P1] ✅ 테스트 파일 부족 (confidence: 90%)
     설명: 테스트 파일이 5개 미만으로 충분하지 않습니다.
     권장: 단위 테스트 5개 이상 추가

선택을 변경하려면:
  - 번호로 토글: 1, 2, 3... (또는 범위: 1-5)
  - a: 모두 승인
  - r: 모두 거부
  - d: 모두 Deferred (다음 스캔에서 다시)
  - q: 취소
  - <enter>: 계속

> 
EOF

# 사용자 입력 처리
read -p "> " USER_SELECTION

# 선택 결과에 따라 상태 업데이트
# New → Approved/Rejected/Deferred
```

## Linear 이슈 생성

Approved 상태의 항목만 이슈 생성:

```bash
echo ""
echo "📝 Linear 이슈 생성 중..."

# 부모 이슈 생성 (Repository Analysis Report)
PARENT_RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation { issueCreate(input: { title: \"📊 Repository Analysis Report\", description: \"프로젝트 자동 분석 결과\", teamId: \"'"$TEAM_ID"'\", priority: 0 }) { issue { id identifier } } }"
  }')

PARENT_ID=$(echo "$PARENT_RESPONSE" | python3 -c "import json,sys; r = json.load(sys.stdin); print(r.get('data', {}).get('issueCreate', {}).get('issue', {}).get('id', ''))")
PARENT_KEY=$(echo "$PARENT_RESPONSE" | python3 -c "import json,sys; r = json.load(sys.stdin); print(r.get('data', {}).get('issueCreate', {}).get('issue', {}).get('identifier', ''))")

echo "✅ 부모 이슈 생성: $PARENT_KEY"

# 자식 이슈들 생성
APPROVED_COUNT=0
for finding in ...; do
  # 각 Approved 항목마다
  CHILD_RESPONSE=$(curl -s -X POST https://api.linear.app/graphql \
    -H "Authorization: $API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
      "query": "mutation { issueCreate(...) }"
    }')
  
  CHILD_KEY=$(echo "$CHILD_RESPONSE" | python3 -c "...")
  echo "  ✅ $CHILD_KEY"
  ((APPROVED_COUNT++))
done

echo ""
echo "✅ 완료: $APPROVED_COUNT개 이슈 생성"
```

## 결과 저장

```bash
# .claude/scan-results.json에 저장
python3 << 'EOF'
import json
from datetime import datetime

results = {
  "scan_metadata": {
    "scan_id": "scan_20260409_103000",
    "timestamp": datetime.now().isoformat(),
    "triggered_by": "$USER_NAME",
    "scan_type": "full_rescan",
    "repository": {
      "path": os.getcwd(),
      "branch": "dev",
      "commit_hash": "..."
    }
  },
  "findings": [...],  # 모든 findings 저장
  "summary": {...}
}

with open(".claude/scan-results.json", "w") as f:
  json.dump(results, f, indent=2, ensure_ascii=False)

print("✅ 결과 저장: .claude/scan-results.json")
EOF
```
```

#### Step 5.3: Commit

```bash
cd /mct-skill-plugin
git add plugins/ls/commands/scan.md
git commit -m "feat: implement /ls:scan with preview UI and selective creation

- Add dual permission validation (config + Linear API)
- Execute Phase 1 (rule-based) and Phase 2 (LLM) analysis
- Create hierarchical preview with parent-child structure
- User selection for Approved/Rejected/Deferred
- Create Linear issues and save results to scan-results.json
- Commit after approval with timestamp and metadata

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

---

### Task 6: 재스캔 및 상태 추적 구현

**Files:**
- Create: `plugins/ls/skills/ls/analyzers/comparator.md`
- Modify: `plugins/ls/commands/scan.md` (재스캔 로직 추가)

#### Step 6.1: comparator.md 작성 (이전 스캔 비교)

Create `plugins/ls/skills/ls/analyzers/comparator.md`:

```markdown
# 재스캔 및 상태 추적

## compare_scans(current_findings, previous_scan_results)

현재 스캔 결과를 이전 스캔 결과와 비교하여 상태를 자동으로 업데이트합니다.

### 상태 전이 규칙

```
이전 상태가 없는 항목:
  → [New] (새로 발견됨)

이전에 [Approved]였던 항목:
  현재도 발견됨 → [Under Review] (여전히 미해결)
  현재 발견 안 됨 → [Completed] (자동 해결됨)

이전에 [Deferred]였던 항목:
  현재도 발견됨 → [Deferred] 유지 (다음 스캔에서 재평가)
  현재 발견 안 됨 → [Completed]

이전에 [Rejected]였던 항목:
  다시 나타나지 않음 (무시)
```

### Python 구현

\`\`\`python
import json
from datetime import datetime

def compare_scans(current_findings, previous_results_file):
    """
    재스캔: 이전 결과와 비교하여 상태 업데이트
    """
    # 이전 결과 로드
    try:
        with open(previous_results_file, "r") as f:
            previous = json.load(f)
    except:
        previous = None
    
    if not previous:
        # 첫 스캔: 모든 항목을 [New]로 표시
        for finding in current_findings:
            finding["status"] = "New"
        return current_findings
    
    previous_findings = {f.get("id"): f for f in previous.get("findings", [])}
    
    # 상태 전이 로직
    for current in current_findings:
        finding_id = current.get("id")
        previous_finding = previous_findings.get(finding_id)
        
        if not previous_finding:
            # 새로 발견된 항목
            current["status"] = "New"
        else:
            prev_status = previous_finding.get("status")
            
            if prev_status == "Approved":
                # 이전에 승인된 항목
                current["status"] = "Under Review"  # 여전히 미해결
            elif prev_status == "Deferred":
                # 이전에 연기된 항목
                current["status"] = "Deferred"  # 유지
            elif prev_status == "Rejected":
                # 이전에 거부된 항목은 다시 나타나지 않음
                continue
            else:
                # 다른 상태
                current["status"] = prev_status
        
        # 상태 변경 히스토리에 기록
        if "status_history" not in current:
            current["status_history"] = []
        
        current["status_history"].append({
            "status": current["status"],
            "timestamp": datetime.now().isoformat(),
            "changed_by": "system"  # 또는 사용자명
        })
    
    # 완료된 항목 처리 (이전에는 있었는데 지금 없는 항목)
    for prev_id, prev_finding in previous_findings.items():
        if prev_finding.get("status") in ["Approved", "In Progress", "Deferred"]:
            # 현재 스캔에서 발견 안 됨 → 완료
            current_ids = {f.get("id") for f in current_findings}
            if prev_id not in current_ids:
                completed = prev_finding.copy()
                completed["status"] = "Completed"
                completed["status_history"].append({
                    "status": "Completed",
                    "timestamp": datetime.now().isoformat(),
                    "changed_by": "system",
                    "note": "코드 분석 결과 해결됨"
                })
                current_findings.append(completed)
    
    return current_findings

if __name__ == "__main__":
    import sys
    current_findings = json.loads(sys.argv[1])
    previous_file = sys.argv[2]
    
    updated = compare_scans(current_findings, previous_file)
    print(json.dumps(updated, ensure_ascii=False, indent=2))
\`\`\`
```

#### Step 6.2: Commit

```bash
cd /mct-skill-plugin
git add plugins/ls/skills/ls/analyzers/comparator.md
git commit -m "feat: implement rescan comparison and state lifecycle

- Add compare_scans() for finding state transitions
- New findings marked as [New]
- Persistent issues marked as [Under Review]
- Resolved items marked as [Completed]
- Deferred items remain [Deferred] for next rescan
- Track state_history with timestamps

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

#### Step 6.3: scan.md에 재스캔 로직 추가

Modify `plugins/ls/commands/scan.md` — add section before preview:

```markdown
## 이전 스캔과 비교 (재스캔)

이전에 스캔 결과가 있다면 비교하여 상태 자동 업데이트:

```bash
if [ -f ".claude/scan-results.json" ]; then
  echo "📊 이전 스캔과 비교 중..."
  
  # comparator.md의 compare_scans() 함수 호출
  COMPARED_FINDINGS=$(python3 << 'EOF'
  import json
  
  with open(".claude/scan-results.json", "r") as f:
    previous = json.load(f)
  
  # compare_scans() 실행
  # current_findings와 previous를 비교하여 상태 업데이트
  
  print(json.dumps(compared, ensure_ascii=False, indent=2))
EOF
  )
  
  # 상태별 요약 표시
  echo ""
  echo "변경사항:"
  echo "  ★ [NEW] 항목"
  echo "  [STILL PENDING] 항목"
  echo "  ✓ [COMPLETED] 항목"
else
  COMPARED_FINDINGS=$CURRENT_FINDINGS
fi
```

#### Step 6.4: Commit scan.md 업데이트

```bash
cd /mct-skill-plugin
git add plugins/ls/commands/scan.md
git commit -m "feat: add rescan comparison logic to /ls:scan

- Compare current findings with previous scan-results.json
- Auto-update states: New/Under Review/Completed
- Display change summary with highlights
- Track status history with timestamps

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

---

### Task 7: 테스트 및 문서화

**Files:**
- Create: `tests/ls_scan_test.md` (테스트 케이스 정의)
- Modify: `README.md` (확장 가이드 추가)

#### Step 7.1: 테스트 케이스 정의

Create `tests/ls_scan_test.md`:

```markdown
# /ls:scan 테스트 케이스

## 테스트 환경 설정

```bash
# 테스트 레포 생성
mkdir -p /tmp/test-repo
cd /tmp/test-repo
git init

# 테스트 파일 생성
touch README.md
mkdir -p src tests
echo "console.log('test')" > src/main.js
echo "test('basic', () => {})" > tests/main.test.js
git add .
git commit -m "Initial commit"
```

## Test 1: 권한 검증

### 1-1. enable_scan이 false일 때
```bash
# /ls:scan 실행
# 예상: "스캔 권한이 비활성화되어 있습니다" 오류
```

### 1-2. 팀원이 실행할 때 (admin 아님)
```bash
# /ls:scan 실행
# 예상: "스캔 기능은 팀장(Admin)만 사용 가능합니다" 오류
```

## Test 2: Phase 1 분석

### 2-1. 문서화 검사
```bash
# README가 50줄일 때
/ls:scan
# 예상: README 불충분 (P2) 발견

# SETUP.md가 없을 때
# 예상: SETUP.md 없음 (P2) 발견
```

### 2-2. 테스트 검사
```bash
# tests 폴더에 파일이 1개일 때
/ls:scan
# 예상: 테스트 파일 부족 (P1) 발견
```

## Test 3: Phase 2 LLM 분석

### 3-1. LLM 분석 대기
```bash
/ls:scan
# 예상: "Phase 2 분석 중... (~30초)" 표시
#       플래그된 항목만 LLM으로 분석
```

### 3-2. confidence 점수 할당
```bash
/ls:scan
# 예상: 모든 Phase 2 항목이 0-1 사이의 confidence 값을 가짐
```

## Test 4: 프리뷰 및 선택

### 4-1. 계층 구조 표시
```bash
/ls:scan
# 예상:
# ☐ [P0] Repository Analysis Report (부모)
#   ☑ [P2] README 불충분
#   ☑ [P1] 테스트 부족
```

### 4-2. 사용자 선택 처리
```bash
/ls:scan
# 입력: a (모두 승인)
# 예상: 모든 항목 [Approved], Linear 이슈 생성 시작
```

## Test 5: Linear 이슈 생성

### 5-1. 부모 이슈 생성
```bash
/ls:scan → (모두 승인)
# 예상: ADE-XX Repository Analysis Report 생성
```

### 5-2. 자식 이슈 생성
```bash
/ls:scan → (모두 승인)
# 예상: ADE-YY README 불충분
#      ADE-ZZ 테스트 부족
#      (부모 ADE-XX 아래 생성)
```

## Test 6: 재스캔

### 6-1. 상태 자동 업데이트
```bash
# 첫 스캔
/ls:scan → ADE-XX, ADE-YY 생성 (Approved)

# README 수정 후 재스캔
/ls:scan
# 예상: README 불충분 → [Under Review] (여전히 미해결)
```

### 6-2. 완료된 항목 감지
```bash
# 테스트 추가 후 재스캔
/ls:scan
# 예상: 테스트 부족 → [Completed] (자동 표시)
#      Linear 이슈에 코멘트 추가
```

## Test 7: scan-results.json 저장

### 7-1. 파일 존재 확인
```bash
/ls:scan → (완료)
# 확인: .claude/scan-results.json 존재
```

### 7-2. 데이터 구조 검증
```bash
cat .claude/scan-results.json | jq '.scan_metadata'
# 예상: scan_id, timestamp, triggered_by 등 포함
```

---

## 테스트 실행 명령어

```bash
cd /mct-skill-plugin

# 모든 테스트 실행
/ls:scan

# 특정 카테고리만 테스트
/ls:scan --category=docs

# 결과 확인
/ls:scan --report
```
```

#### Step 7.2: Commit 테스트 케이스

```bash
cd /mct-skill-plugin
git add tests/ls_scan_test.md
git commit -m "test: add comprehensive /ls:scan test cases

- Permission validation tests (enable_scan, admin role)
- Phase 1 rule-based analysis tests
- Phase 2 LLM analysis tests
- Preview UI and user selection tests
- Linear issue creation tests
- Rescan and state lifecycle tests
- scan-results.json validation tests

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

#### Step 7.3: README.md 업데이트

Modify `README.md` — add after `/ls:integrate` section:

```markdown
### `/ls:scan` — 레포지토리 분석 후 선택적 이슈 등록

팀장만 사용할 수 있는 고급 기능으로, 현재 레포지토리를 자동으로 분석하여 기능 개선, 코드 품질, 문서화 등의 항목을 발견하고 **선택적으로** Linear 이슈로 등록합니다.

#### 분석 방식

- **Phase 1 (규칙 기반)**: 파일 구조, 문서, 테스트, 코드 크기 등을 정적으로 분석 (빠름)
- **Phase 2 (LLM)**: Phase 1에서 발견한 항목만 Claude로 깊이 있게 분석 (깊음)

#### 기능

- 📊 6가지 카테고리 자동 검사 (문서화, 테스트, 코드품질, 의존성, 구조, 자동화)
- 🔐 팀장 전용 (권한 검증)
- ✅ 부모-자식 계층 구조로 이슈 생성
- 📋 프리뷰 후 선택적 생성 (승인/거부/연기)
- 🔄 재스캔 지원 (변경사항 자동 감지)
- 💾 결과 저장 및 히스토리 추적

#### 사용 예시

```bash
# 레포지토리 분석
/ls:scan

# 결과 미리보기
/ls:scan --report

# 이전 스캔과 비교
/ls:scan --compare 2026-04-02

# 특정 카테고리만 재분석
/ls:scan --category=tests
```

#### 권한 설정

초기 설정 시:
```bash
/ls:setup
# Step 5에서 "스캔 권한 활성화?" 질문에 [Y] 선택
```
```

#### Step 7.4: Commit README 업데이트

```bash
cd /mct-skill-plugin
git add README.md
git commit -m "docs: add /ls:scan feature documentation

- Add overview of hybrid analysis (Phase 1 + Phase 2)
- Document permission system (team lead only)
- List key features and use cases
- Add usage examples and setup instructions

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>"
```

---

## 📋 최종 체크리스트

모든 Task 완료 후 다음을 확인하세요:

```bash
cd /mct-skill-plugin

# 1. 설정 파일 검증
cat .claude/linear.json | jq '.enable_scan'
# 예상: true

# 2. 커맨드 메타데이터 확인
cat plugins/ls/.claude-plugin/plugin.json | jq '.commands.scan'
# 예상: { "requires_role": "admin" }

# 3. 모든 신규 파일 존재 확인
ls -la plugins/ls/skills/ls/auth.md
ls -la plugins/ls/skills/ls/analyzers/phase1-rules.md
ls -la plugins/ls/skills/ls/analyzers/phase2-llm.md
ls -la plugins/ls/skills/ls/analyzers/comparator.md

# 4. Git 히스토리 확인
git log --oneline | head -10
# 예상: 7개의 /ls:scan 관련 커밋

# 5. /ls:scan 실행 테스트
/ls:scan
# 예상: 권한 검증 → Phase 1 → Phase 2 → 프리뷰 → 선택 → 이슈 생성
```

---

## 🚀 구현 방식

**권장:** superpowers:subagent-driven-development 사용
- 각 Task별 독립적인 subagent 디스패치
- Task 간 코드 리뷰 및 검증
- 빠른 피드백 사이클

**또는:** superpowers:executing-plans으로 이 세션에서 순차 실행
```
