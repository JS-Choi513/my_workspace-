# Workspace

## 새 세션 시작 시 필독 순서

1. **이 파일** — 구조·규칙 파악
2. **`handoff/current-state.md`** — 현재 진행 상태, 다음 작업, 미처리 이슈
3. **해당 프로젝트 `CLAUDE.md`** — 아키텍처·명령어·설계 결정

> 세션 도중 배경 정보가 필요하면 아래 컨텍스트 파일 참조.

---

## 현재 활성 프로젝트

| 프로젝트 | 경로 | 상태 |
|---------|------|------|
| GPU 서버 출고 전 검수 자동화 | `projects/inspection-system/` | v2 리팩토링 진행 중 |

---

## 폴더 구조

| 폴더 | 역할 | 주요 파일 |
|------|------|----------|
| `context/` | 배경·환경·용어 (원칙적으로 불변, 갱신 조건은 아래 참조) | about-me.md, glossary.md, infra-environment.md, target-servers.md |
| `rules/` | 워크스페이스 공통 코딩·문서 규칙 | api.md, writing.md |
| `projects/` | 프로젝트별 소스코드 | inspection-system/ |
| `handoff/` | 세션 인수인계 메모 | current-state.md |
| `templates/` | 출력 구조 고정용 양식 | inspection-report-brief.md |
| `outputs/` | 최종 결과물 | — |

---

## 컨텍스트 파일

| 파일 | 내용 |
|------|------|
| `context/about-me.md` | 사용자 역할·배경 |
| `context/glossary.md` | 시스템 용어 정의 |
| `context/infra-environment.md` | 검수 시스템 호스트·NFS·네트워크·대상 서버 SSH 접근 |
| `context/target-servers.md` | 검수 대상 서버 제품군·스펙·가속기 (환경·SSH → infra-environment.md) |
| `rules/api.md` | API·코드 작성 규칙 (프로젝트별 `.claude/rules/` 우선) |
| `rules/writing.md` | 커밋·문서 작성 규칙 |

---

## 글로벌 규칙

### CLI
- 모든 명령은 `rtk <cmd>` — 훅이 자동 변환하므로 별도 처리 불필요

### Git / PR

| 구분 | 브랜치 정책 |
|------|-----------|
| **개인 solo repo** | `main` 직접 push 허용. 브랜치 불필요 |
| **팀 공유 repo** | `main` 직접 push 금지. 브랜치 필수: `feature/`, `fix/`, `chore/` |

현재 `inspection-system`은 팀 공유 repo → main 직접 push 금지. 브랜치 필수.  
완료 기준: `pytest` + `ruff` 통과 → commit → PR 생성. merge는 사용자 결정.

### context 파일 갱신 정책

`context/` 파일은 원칙적으로 불변이나 아래 조건에서 갱신:

| 파일 | 갱신 조건 |
|------|----------|
| `about-me.md` | 역할·기술스택 변경 시 |
| `glossary.md` | 신규 용어 추가 또는 기존 정의 변경 시 |
| `infra-environment.md` | 서버 IP·포트·Docker 구성 변경 시 |
| `target-servers.md` | 신규 제품군 추가 또는 단종 변경 시 |

갱신 주체: 사용자 또는 사용자 지시를 받은 에이전트. 에이전트 독단 갱신 금지.

### 규칙 우선순위

충돌 시 아래 순서대로 적용 (높을수록 우선):

```
1. 프로젝트 .claude/rules/*.md   ← 가장 구체적, 항상 우선
2. 프로젝트 CLAUDE.md            ← 프로젝트 아키텍처·명령어
3. 워크스페이스 rules/*.md       ← 프로젝트 규칙 없는 영역에만 적용
4. 글로벌 ~/.claude/CLAUDE.md    ← 기본값, 가장 광범위
```

**판단 기준**: 같은 주제(예: API 작성, Git 브랜치)에 대해 여러 계층에 규칙이 있으면 가장 구체적인 계층(숫자 낮은 쪽)을 따름. 규칙이 존재하지 않는 영역은 상위 계층으로 fallback.

**예외**: 보안 규칙(`security.md`)과 완료 워크플로우(pytest + ruff → PR)는 계층 무관 항상 적용.
