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

```python
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
```
