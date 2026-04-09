# 재스캔 및 상태 추적

## compare_scans(current_findings, previous_scan_results)

현재 스캔 결과를 이전 스캔 결과와 비교하여 상태를 자동으로 업데이트합니다.

### 상태 전이 규칙

이전 상태가 없는 항목: → [New]
이전에 [Approved]였던 항목이 현재도 발견됨: → [Under Review]
이전에 [Approved]였던 항목이 현재 발견 안 됨: → [Completed]
이전에 [Deferred]였던 항목: → [Deferred] (유지)
이전에 [Rejected]였던 항목: (다시 나타나지 않음)

### Python 구현

```python
import json
from datetime import datetime

def compare_scans(current_findings, previous_results_file):
    """
    재스캔: 이전 결과와 비교하여 상태 업데이트
    """
    try:
        with open(previous_results_file, "r") as f:
            previous = json.load(f)
    except:
        previous = None
    
    if not previous:
        for finding in current_findings:
            finding["status"] = "New"
        return current_findings
    
    previous_findings = {f.get("id"): f for f in previous.get("findings", [])}
    
    for current in current_findings:
        finding_id = current.get("id")
        previous_finding = previous_findings.get(finding_id)
        
        if not previous_finding:
            current["status"] = "New"
        else:
            prev_status = previous_finding.get("status")
            
            if prev_status == "Approved":
                current["status"] = "Under Review"
            elif prev_status == "Deferred":
                current["status"] = "Deferred"
            elif prev_status == "Rejected":
                continue
            else:
                current["status"] = prev_status
        
        if "status_history" not in current:
            current["status_history"] = []
        
        current["status_history"].append({
            "status": current["status"],
            "timestamp": datetime.now().isoformat(),
            "changed_by": "system"
        })
    
    for prev_id, prev_finding in previous_findings.items():
        if prev_finding.get("status") in ["Approved", "In Progress", "Deferred"]:
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
```
