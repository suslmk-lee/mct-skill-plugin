# Phase 2: LLM 상세 분석

Phase 1에서 플래그된 항목(🟡🔴)만 Claude API로 깊이 있게 분석합니다.

## analyze_phase2_llm(flagged_findings, repo_path, api_key)

Phase 1에서 발견한 항목 중 🟡(Warning) 또는 🔴(Alert)만 선택하여:
1. 실제 코드/문서 읽기
2. 근본 원인 파악
3. 구체적 개선 방안 제시
4. confidence 점수 (0-1) 할당

### Python 구현

```python
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
        "의존성": f"""
프로젝트 '{repo_path}'의 의존성 이슈를 분석해주세요:

이슈: {title}
설명: {details}

다음을 제공해주세요:
1. 문제점 분석 (왜 이것이 문제인가)
2. 권장 해결 방안
3. 예상 영향도
4. 신뢰도 점수

형식:
```json
{{
  "problem_analysis": "...",
  "recommended_solution": "...",
  "estimated_impact": "...",
  "confidence": 0.85
}}
```
""",
        "구조": f"""
프로젝트 '{repo_path}'의 구조 이슈를 분석해주세요:

이슈: {title}
설명: {details}

다음을 제공해주세요:
1. 현재 구조의 문제점
2. 개선된 폴더 구조 제안
3. 마이그레이션 전략
4. 신뢰도 점수

형식:
```json
{{
  "current_problems": "...",
  "proposed_structure": "...",
  "migration_strategy": "...",
  "confidence": 0.82
}}
```
""",
        "자동화": f"""
프로젝트 '{repo_path}'의 자동화 이슈를 분석해주세요:

이슈: {title}
설명: {details}

다음을 제공해주세요:
1. 현재 자동화 상태
2. 필요한 자동화 (CI/CD, hooks 등)
3. 구현 순서
4. 신뢰도 점수

형식:
```json
{{
  "current_automation": "...",
  "required_automation": ["...", "..."],
  "implementation_order": ["...", "..."],
  "confidence": 0.90
}}
```
""",
    }
    
    return prompts.get(category, f"""
이슈를 분석해주세요:

이슈: {title}
설명: {details}

다음을 제공해주세요:
1. 근본 원인
2. 개선 방안
3. 신뢰도 점수 (0-1)

형식:
```json
{{
  "root_cause": "...",
  "improvement": "...",
  "confidence": 0.80
}}
```
""")

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
```
