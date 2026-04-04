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
| `context/` | 배경·환경·용어 (세션 간 불변) | about-me.md, glossary.md, infra-environment.md, target-servers.md |
| `rules/` | 워크스페이스 공통 코딩·문서 규칙 | api.md, writing.md, inspection-report-brief.md |
| `projects/` | 프로젝트별 소스코드 | inspection-system/ |
| `handoff/` | 세션 인수인계 메모 | current-state.md |
| `templates/` | 출력 구조 고정용 양식 | — |
| `outputs/` | 최종 결과물 | — |

---

## 컨텍스트 파일

| 파일 | 내용 |
|------|------|
| `context/about-me.md` | 사용자 역할·배경 |
| `context/glossary.md` | 시스템 용어 정의 |
| `context/infra-environment.md` | 검수 시스템 호스트·NFS·네트워크 구성 |
| `context/target-servers.md` | 검수 대상 서버 제품군·환경 |
| `rules/api.md` | API·코드 작성 규칙 (프로젝트별 `.claude/rules/` 우선) |
| `rules/writing.md` | 커밋·문서 작성 규칙 |

---

## 글로벌 규칙

### CLI
- 모든 명령은 `rtk <cmd>` — 훅이 자동 변환하므로 별도 처리 불필요

### Git / PR
- `main` 직접 push 금지
- 브랜치 명명: `feature/`, `fix/`, `chore/`
- 완료 기준: `pytest` + `ruff` 통과 → PR 생성 → 사용자 merge 결정

### 우선순위
- 프로젝트별 규칙은 해당 프로젝트 `CLAUDE.md` 및 `.claude/rules/` 우선
- 워크스페이스 `rules/`는 프로젝트 규칙이 없는 영역에 적용
