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

```python
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
```

### 호출 방법

Bash에서:
```bash
RESULT=$(python3 check_scan_permission.py "$TEAM_ID" "$API_KEY")
IS_ADMIN=$(echo "$RESULT" | python3 -c "import json,sys; print(json.load(sys.stdin)['is_admin'])")
USER_NAME=$(echo "$RESULT" | python3 -c "import json,sys; print(json.load(sys.stdin)['user_name'])")
ROLE=$(echo "$RESULT" | python3 -c "import json,sys; print(json.load(sys.stdin)['role'])")

if [ "$IS_ADMIN" != "True" ]; then
  echo "❌ 스캔 기능은 팀장(Admin)만 사용 가능합니다."
  echo "현재 사용자: $USER_NAME (권한: $ROLE)"
  exit 1
fi
```
