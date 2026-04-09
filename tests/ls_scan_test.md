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
- /ls:scan 실행
- 예상: "스캔 권한이 비활성화되어 있습니다" 오류

### 1-2. 팀원이 실행할 때 (admin 아님)
- /ls:scan 실행
- 예상: "스캔 기능은 팀장(Admin)만 사용 가능합니다" 오류

## Test 2: Phase 1 분석

### 2-1. 문서화 검사
- README가 50줄일 때
- 예상: README 불충분 (P2) 발견
- SETUP.md가 없을 때
- 예상: SETUP.md 없음 (P2) 발견

### 2-2. 테스트 검사
- tests 폴더에 파일이 1개일 때
- 예상: 테스트 파일 부족 (P1) 발견

## Test 3: Phase 2 LLM 분석

### 3-1. LLM 분석 대기
- /ls:scan 실행
- 예상: "Phase 2 분석 중... (~30초)" 표시, 플래그된 항목만 LLM으로 분석

### 3-2. confidence 점수 할당
- /ls:scan 실행
- 예상: 모든 Phase 2 항목이 0-1 사이의 confidence 값을 가짐

## Test 4: 프리뷰 및 선택

### 4-1. 계층 구조 표시
- /ls:scan 실행
- 예상: ☐ [P0] Repository Analysis Report (부모)
         ☑ [P2] README 불충분
         ☑ [P1] 테스트 부족

### 4-2. 사용자 선택 처리
- /ls:scan 실행
- 입력: a (모두 승인)
- 예상: 모든 항목 [Approved], Linear 이슈 생성 시작

## Test 5: Linear 이슈 생성

### 5-1. 부모 이슈 생성
- /ls:scan → (모두 승인)
- 예상: ADE-XX Repository Analysis Report 생성

### 5-2. 자식 이슈 생성
- /ls:scan → (모두 승인)
- 예상: ADE-YY README 불충분, ADE-ZZ 테스트 부족 (부모 ADE-XX 아래 생성)

## Test 6: 재스캔

### 6-1. 상태 자동 업데이트
- 첫 스캔: /ls:scan → ADE-XX, ADE-YY 생성 (Approved)
- README 수정 후 재스캔: /ls:scan
- 예상: README 불충분 → [Under Review] (여전히 미해결)

### 6-2. 완료된 항목 감지
- 테스트 추가 후 재스캔: /ls:scan
- 예상: 테스트 부족 → [Completed] (자동 표시)

## Test 7: scan-results.json 저장

### 7-1. 파일 존재 확인
- /ls:scan → (완료)
- 확인: .claude/scan-results.json 존재

### 7-2. 데이터 구조 검증
- `cat .claude/scan-results.json | jq '.scan_metadata'`
- 예상: scan_id, timestamp, triggered_by 등 포함
