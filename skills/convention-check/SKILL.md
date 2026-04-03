---
name: convention-check
description: "프로젝트 컨벤션 위배 사항을 검사하고 보고"
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion
user-invocable: true
---

## Prerequisites

### 설정 파일
- **`.convention-check.json`** (프로젝트 루트): 적용할 컨벤션 목록을 저장한다.
- 이 파일이 없으면 기본값 (`default-conventions` + `pagenation`)으로 동작한다.

### `--init` (컨벤션 선택)

`$ARGUMENTS`가 `--init`이면 아래 절차를 실행하고 종료한다:

1. **사용 가능한 컨벤션 목록 제시**: 플러그인 내 컨벤션 스킬과 프로젝트의 CLAUDE.md를 탐색하여 목록을 구성한다.

   > "어떤 컨벤션을 적용할까요? (복수 선택 가능, 쉼표 구분)"
   >
   > **플러그인 내장:**
   > 1. `default-conventions` — 에러 처리, VO 패턴, 트랜잭션 관리
   > 2. `pagenation` — 커서 기반 페이지네이션
   >
   > **프로젝트:**
   > 3. `CLAUDE.md` — 프로젝트 아키텍처 및 레이어 컨벤션
   >
   > 예: `1,2,3` (전체) 또는 `1` (에러 처리만)

2. **유저 선택 수집**: `AskUserQuestion`으로 선택을 받는다.

3. **설정 파일 생성**: 선택 결과를 `.convention-check.json`으로 저장한다.
   ```json
   {
     "conventions": [
       { "name": "default-conventions", "source": "plugin", "skill": "minmo-s-harness:default-conventions" },
       { "name": "pagenation", "source": "plugin", "skill": "minmo-s-harness:pagenation" },
       { "name": "CLAUDE.md", "source": "project", "path": "CLAUDE.md" }
     ]
   }
   ```

4. **결과 보고**:
   > "`.convention-check.json` 생성 완료. 적용 컨벤션: [선택 목록]"

### `--doctor` (상태 진단)

`$ARGUMENTS`가 `--doctor`이면 아래 항목을 점검하고 결과를 보고한 뒤 종료한다:

```markdown
## Convention Check — Doctor

| 항목 | 상태 | 비고 |
|------|------|------|
| .convention-check.json | OK / MISSING | 없으면 기본값 사용 |
| default-conventions 스킬 | OK / MISSING | 플러그인 스킬 확인 |
| pagenation 스킬 | OK / MISSING | 플러그인 스킬 확인 |
| CLAUDE.md | OK / MISSING | 프로젝트 루트 확인 |
| 적용 컨벤션 목록 | [목록] | 현재 설정 표시 |
```

---

## Execution

### 설정 파일 로드

1. 프로젝트 루트에서 `.convention-check.json`을 읽는다.
2. 없으면 기본값으로 진행: `default-conventions` + `pagenation`

### 컨벤션 검사

설정된 각 컨벤션에 대해 위배 사항을 검사하고 보고한다:

- **`source: "plugin"`**: 해당 스킬의 내용을 기준으로 코드를 검사한다.
- **`source: "project"`**: 해당 파일의 내용을 기준으로 코드를 검사한다.
