# `/ls:scan` 확장 설계

**작성일:** 2026-04-09  
**팀:** Linear Skill (ls)  
**상태:** 설계 완료 & 승인

---

## 📋 개요

현재 작업 중인 레포지토리를 **자동으로 분석**하여 기능 개선, 코드 품질, 문서화, 테스트 커버리지 등의 항목을 발견하고 **선택적으로 Linear 이슈로 등록**하는 기능입니다.

**핵심 특징:**
- 하이브리드 분석 (규칙 기반 + LLM)
- 단계적 프리뷰 후 선택적 생성
- 재스캔 지원 (변경 사항 추적)
- 팀장 전용 (권한 관리)
- 상태 라이프사이클 추적

---

## 🎯 목표

1. **발견** — 개발 중 놓치기 쉬운 개선 항목을 자동으로 발견
2. **우선순위화** — 발견된 항목을 P0/P1/P2로 분류
3. **선택성** — 팀장이 검토 후 필요한 것만 이슈화
4. **추적성** — 프로젝트 변화에 따라 상태 자동 업데이트
5. **투명성** — 팀원이 언제든 현재 분석 결과 조회 가능

---

## 🔐 권한 관리 시스템

### 설정 단계 (`/ls:setup`)

1. 기존 설정 (API 키, 팀 선택, 상태 매핑) 완료 후
2. **팀장 전용 질문:**
   ```
   Linear에서 당신의 역할: Admin (ADE 팀)
   
   이 팀에서 저장소 스캔 기능(/ls:scan)을
   활성화하시겠습니까?
   - [Y] 활성화 (팀장만 사용 가능)
   - [N] 비활성화 (다른 팀원만 사용)
   ```
3. Y 선택 시:
   - `.claude/linear.json`에 `enable_scan: true` 저장
   - `scan_admin_user: "kim@company.com"` 기록
   - `plugin.json`에 `/ls:scan` 커맨드 추가 (메타데이터: `requires_role: "admin"`)

### 실행 단계 (`/ls:scan`)

**검증 1: 설정 파일 확인**
```
if .claude/linear.json의 enable_scan ≠ true:
  → "스캔 권한이 비활성화되어 있습니다."
  → "팀장에게 /ls:setup으로 활성화를 요청하세요."
  → 실행 중지
```

**검증 2: Linear 사용자 역할 확인**
```bash
# Linear API 호출
query {
  viewer {
    teamMembership(teamId: "LINEAR_TEAM_ID") {
      role  # "admin" 또는 "member"
    }
  }
}

if role ≠ "admin":
  → "스캔 기능은 팀장(Admin)만 사용 가능합니다."
  → "현재 사용자: {user} (권한: {role})"
  → 실행 중지
```

**커맨드 표시:**
- 팀장: `/ls:scan` ✅ 보임
- 팀원: `/ls:scan` 👻 숨겨짐 + 타이핑하면 "권한 없음" 오류

---

## 📊 분석 아키텍처

### Phase 1: 규칙 기반 분석 (빠름)

정적 패턴으로 빠르게 스캔:

| 카테고리 | 검사 항목 | 플래그 기준 |
|---------|---------|-----------|
| **문서화** | README 존재/길이, SETUP.md, API 문서 | README < 100줄 = 🟡 |
| **테스트** | test/ 폴더, 테스트 파일 비율, 주요 파일 미테스트 | 커버리지 < 50% = 🔴 |
| **코드품질** | 파일 크기, 함수 길이, 중첩 깊이, 주석 부재 | 파일 > 500줄 = 🟡 |
| **의존성** | package.json 분석, 미사용 패키지, 버전 정보 | outdated = 🔴 |
| **구조** | 폴더 계층, 파일 분류, 모듈 응집도 | util에 비즈니스로직 = 🟡 |
| **자동화** | CI/CD 설정, pre-commit hooks, 버전관리 | .github/workflows 없음 = 🟡 |

**플래그:**
- 🟢 **Good** — 문제 없음 (이슈 미생성)
- 🟡 **Warning** — 개선 권장 (P2 이슈 후보)
- 🔴 **Alert** — 즉시 개선 필요 (P0/P1 이슈 후보)

**제외 대상:**
- `.gitignore` 패턴 자동 읽기 → 제외
- 기본값: `node_modules`, `.git`, `dist`, `build`, `.env*`, `*.log` 등

### Phase 2: LLM 상세 분석 (깊음)

Phase 1에서 🔴/🟡 플래그된 항목만 처리:

1. 실제 코드/문서 읽기
2. 근본 원인 파악
3. 구체적 개선 방안 제시
4. confidence 점수 할당 (0-1)

**예시:**
```
Phase 1 발견: 🟡 AuthService.ts > 500줄
Phase 2 분석:
  - "인증/인가/로깅 3가지 책임을 함"
  - "클래스 분리 추천"
  - confidence: 0.92
```

---

## 📋 이슈 생성 구조

### 계층 구조 (부모-자식)

```
[부모] 📊 Repository Analysis Report (P0, 상태: Approved)
  ├── [부모] 📚 문서화 개선 (P2)
  │   ├── README 확장 필요
  │   ├── API 문서 작성
  │   └── 주석 추가 (src/auth.ts)
  │
  ├── [부모] ✅ 테스트 커버리지 (P1)
  │   ├── 단위 테스트 추가 (src/auth.ts)
  │   └── 통합 테스트 작성 (api.ts)
  │
  ├── [부모] 🏗️ 코드품질 (P1)
  │   ├── AuthService 함수 분리 (confidence: 92%)
  │   └── 순환 참조 제거
  │
  └── [부모] 🔐 보안 (P0)
      └── 의존성 업데이트 (lodash, express)
```

### 사용자 선택 (프리뷰)

분석 후 프리뷰 화면에서 각 항목별로:
- ☑ 승인 → [Approved] 상태, Linear 이슈 생성
- ☐ 거부 → [Rejected] 상태, 스킵
- ◻ 연기 → [Deferred] 상태, 다음 스캔에서 다시 물어보기

---

## 🔄 상태 라이프사이클

### 상태 태그 (6가지)

```
[New] ──→ [Under Review] ──→ [Approved] ──→ [In Progress] ──→ [Completed]
   ↓
[Rejected]

[Deferred] ──→ (다음 스캔에서 재평가)
```

**상태별 의미:**
- **[New]** — 스캔에서 새로 발견된 항목 (사용자 선택 필요)
- **[Under Review]** — 이전에는 있었던 항목이 재스캔에서 여전히 발견됨 (미해결)
- **[Approved]** — 사용자가 승인, Linear 이슈 생성됨
- **[Rejected]** — 사용자가 거부, 다시 나타나지 않음
- **[Deferred]** — 사용자가 "나중에" 선택, 다음 스캔에서 다시 표시
- **[In Progress]** — 팀원이 해당 Linear 이슈에서 작업 중
- **[Completed]** — 코드 리뷰 완료 또는 수동 표시

### 재스캔 시 동작

**상황 1: 이전에 있던 항목이 여전히 발견됨**
```
상태: [Approved] → [Under Review]로 변경
신호: "여전히 미해결 상태"
조치: 우선순위 재검토 필요
```

**상황 2: 이전 항목이 사라짐 (해결됨)**
```
상태: [Approved|In Progress] → [Completed]로 자동 변경
조치: Linear 이슈에 자동 코멘트 "코드 분석 결과 해결됨"
```

**상황 3: 새로운 항목 발견**
```
상태: [New]
신호: 프리뷰에서 ★ [NEW] 표시
조치: 사용자 승인 필요
```

---

## 📁 데이터 저장

### `.claude/scan-results.json`

프로젝트별 스캔 결과를 타임스탐프와 함께 저장:

```json
{
  "scan_metadata": {
    "scan_id": "scan_20260409_103000",
    "timestamp": "2026-04-09T10:30:00Z",
    "triggered_by": "kim@company.com",
    "scan_type": "full_rescan",
    "previous_scan_id": "scan_20260408_140000",
    "repository": {
      "path": "/current/repo",
      "branch": "dev",
      "commit_hash": "a0d8006"
    }
  },
  "findings": [
    {
      "id": "finding_001",
      "category": "테스트",
      "title": "단위 테스트 커버리지 부족 (10%)",
      "severity": "P1",
      "status": "Approved",
      "phase": "1",
      "details": "src/auth.ts에 테스트 없음",
      "confidence": 0.95,
      "created_at": "2026-04-09T10:30:00Z",
      "created_scan_id": "scan_20260409_103000",
      "last_updated": "2026-04-09T10:30:00Z",
      "linear_issue_key": "ADE-27",
      "status_history": [
        {
          "status": "New",
          "timestamp": "2026-04-09T10:30:00Z",
          "changed_by": "kim@company.com"
        },
        {
          "status": "Approved",
          "timestamp": "2026-04-09T10:35:00Z",
          "changed_by": "kim@company.com"
        }
      ]
    }
  ],
  "summary": {
    "total_findings": 8,
    "new_findings": 2,
    "still_pending": 3,
    "completed": 2,
    "by_status": {
      "New": 0,
      "Under Review": 1,
      "Approved": 3,
      "Rejected": 1,
      "Deferred": 1,
      "In Progress": 1,
      "Completed": 2
    },
    "by_severity": {
      "P0": 1,
      "P1": 3,
      "P2": 4
    }
  }
}
```

**저장 위치:**
- `.claude/scan-results.json` (프로젝트별)
- `.gitignore`에 등록 (보안)

---

## 💻 커맨드 인터페이스

### 기본 사용

```bash
# 전체 분석 (Phase 1 + Phase 2)
/ls:scan

# 특정 카테고리만 재분석
/ls:scan --category=tests
/ls:scan --category=docs

# 과거 Deferred 항목도 표시
/ls:scan --show-deferred

# 완료된 항목도 표시
/ls:scan --show-completed
```

### 결과 조회

```bash
# 현재 상태 요약
/ls:scan --report

# 모든 스캔 히스토리
/ls:scan --history

# 특정 날짜와 비교
/ls:scan --compare 2026-04-02

# 항목 상태 변경
/ls:scan --update-status finding_001=Completed
```

### 권한 관리 (팀장만)

```bash
# 스캔 권한 재설정
/ls:setup --reset-scan

# 스캔 권한 있는 사용자 목록
/ls:scan --list-admins
```

---

## 🔄 사용자 경험 흐름

### 시나리오 1: 팀장이 첫 스캔 실행

```
팀장 김철수) /ls:scan

✅ 검증 1: enable_scan=true ✓
✅ 검증 2: Linear admin 역할 ✓

Phase 1 분석 중...
[████████░░] 80%

Phase 2 분석 중 (3개 항목)...
🔄 AuthService 리팩토링
🔄 테스트 커버리지 분석
(약 30초)

=== 최종 프리뷰 ===

★ [NEW] 📚 README 확장 (P2)
   이전: [Deferred] → 이제: [New]
   설명: API 명세 문서 부재
   ☐ 승인 ☐ 거부 ☐ 연기

✅ [CONFIRMED] ✅ 테스트 커버리지 (P1)
   confidence: 95%
   ☑ 승인 ☐ 거부 ☐ 연기

선택 후 엔터 (a=모두, r=모두거부, q=취소)
> a

✅ 8개 이슈 생성 완료!
  - ADE-25: README 확장
  - ADE-26: 테스트 커버리지
  ...
```

### 시나리오 2: 팀원이 스캔 시도

```
팀원 이순신) /ls:scan

❌ 권한 없음

스캔 기능은 팀장(Admin)만 사용 가능합니다.
- 현재 사용자: lee@company.com (권한: member)

팀장에게 요청하세요: kim@company.com (팀장)
```

### 시나리오 3: 일주일 후 팀장이 재스캔

```
팀장 김철수) /ls:scan

✅ 검증 통과

=== 이전 스캔과 비교 (7일 전) ===

✓ [COMPLETED] 📚 README 확장 (ADE-25)
  → 코드 리뷰 완료, 병합됨

[STILL PENDING] ✅ 테스트 커버리지 (ADE-26)
  → 상태: In Progress (lee@company.com)

★ [NEW] 🔐 보안 의존성 업데이트 (P0)
  → 새로 발견됨 (confidence: 96%)

[DEFERRED] 🏗️ AuthService 리팩토링
  → 여전히 미해결, 상태: Deferred

선택:
> a (새 항목만 생성)

✅ 1개 이슈 생성: ADE-29 (보안 의존성)
```

---

## 📁 프로젝트 구조

완성된 파일 구조:

```
mct-plugin/
├── README.md                        # 프로젝트 문서 (업데이트됨)
├── SETUP.md                         # 설정 가이드
├── .gitignore                       # Git 제외 파일 (업데이트됨)
│
├── docs/superpowers/                # 설계 & 계획 문서 (git 추적 ✓)
│   ├── specs/
│   │   └── 2026-04-09-ls-scan-expansion-design.md
│   └── plans/
│       └── 2026-04-09-ls-scan-implementation.md
│
├── tests/
│   └── ls_scan_test.md              # 테스트 케이스 (완성됨)
│
├── .claude/
│   ├── linear.json                  # 프로젝트 설정 (git 제외)
│   ├── linear.json.template         # 설정 템플릿
│   ├── linear.json.example          # 설정 예시
│   ├── config.local.json            # 로컬 설정 (git 제외)
│   ├── settings.local.json          # 로컬 설정 (git 제외)
│   └── scan-results.json            # 스캔 결과 (동적 생성, git 제외)
│
├── .claude-plugin/
│   └── marketplace.json             # 플러그인 마켓플레이스 인덱스
│
└── plugins/ls/
    ├── .claude-plugin/
    │   └── plugin.json              # [수정] requires_role 메타데이터 추가 ✓
    │
    ├── commands/                    # 커맨드 정의 (9개)
    │   ├── setup.md                 # [수정] 팀장 권한 설정 추가 ✓
    │   ├── start.md
    │   ├── list.md
    │   ├── done.md
    │   ├── pr.md
    │   ├── scan.md                  # [신규] 완성됨 ✓
    │   ├── status.md
    │   ├── sub.md
    │   └── integrate.md
    │
    └── skills/ls/
        ├── SKILL.md
        ├── auth.md                  # [신규] 권한 검증 로직 ✓
        └── analyzers/               # [신규] 분석 모듈들 ✓
            ├── phase1-rules.md      # Phase 1 규칙 기반 분석 ✓
            ├── phase2-llm.md        # Phase 2 LLM 상세 분석 ✓
            └── comparator.md        # 재스캔 & 상태 추적 ✓
```

### .gitignore 정책

```
# Git 추적 O
✓ docs/superpowers/            # 설계 & 계획 문서

# Git 제외
✗ .claude/linear.json          # 프로젝트별 설정
✗ .claude/config.local.json    # 로컬 설정
✗ .claude/settings.local.json  # 로컬 설정
✗ .claude/scan-results.json    # 스캔 결과 (동적 생성)
```

---

## 🚀 구현 계획

1. **Phase 1**: 권한 관리 시스템 구현 (`/ls:setup` 수정)
2. **Phase 2**: 규칙 기반 분석 구현 (Phase 1 로직)
3. **Phase 3**: LLM 상세 분석 구현 (Phase 2 로직)
4. **Phase 4**: 프리뷰 UI 구현 (선택적 생성)
5. **Phase 5**: 재스캔 & 상태 추적 구현
6. **Phase 6**: 테스트 & 문서화

---

## ✅ 검증 체크리스트

- [x] 권한 관리 (팀장 전용)
- [x] 단계적 분석 (규칙 + LLM)
- [x] 선택적 생성 (프리뷰)
- [x] 상태 추적 (6가지 태그)
- [x] 재스캔 지원 (변경사항 추적)
- [x] 커맨드 은닉 (권한 없으면 표시 안됨)
- [x] 데이터 저장 (scan-results.json)

---

**설계 완료 & 승인됨** ✅
