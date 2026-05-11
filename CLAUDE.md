# Workspace

멀티프로젝트 컨테이너. 각 프로젝트는 독립된 git repo로 `projects/` 하위에서 관리.
워크스페이스 하네스는 **프로젝트 인덱스 + 프로젝트 중립 공통 규칙**만 둔다.

## 새 세션 시작 시 필독 순서

1. **이 파일** — 활성 프로젝트 목록·공통 규칙
2. **`handoff/current-state.md`** — 워크스페이스 인덱스 (각 프로젝트 handoff 위치)
3. **작업 대상 프로젝트의 `CLAUDE.md` + `handoff/`** — 실제 진행 상태·아키텍처·명령어

---

## 활성 프로젝트

| 프로젝트 | 경로 | 언어·스택 | 상태 | repo |
|---------|------|----------|------|------|
| inspection-system | `projects/inspection-system/` | Python (FastAPI + Celery + PostgreSQL) | v2 리팩토링 진행 중 | 팀 공유 (DEEPGadget/inspection-system) |
| p2p-perf-monitor | `projects/p2p-perf-monitor/` | Python (FastAPI) + SvelteKit | Phase 4 진행 중 | 팀 공유 (DEEPGadget/p2p-perf-monitor) |
| gadgetron | `projects/gadgetron/` | Rust + npm (web UI) | 외부 clone (하네스 미정비) | 외부 |

각 프로젝트는 **자체 git repo**. `projects/`는 워크스페이스 git에서 ignore (`.gitignore` 참조).

---

## 폴더 구조

| 폴더 | 역할 |
|------|------|
| `context/` | 워크스페이스 공유 컨텍스트 (여러 프로젝트가 참조하는 사양·환경) |
| `rules/` | 워크스페이스 공통 코딩·문서 규칙 (언어·도구 중립) |
| `projects/` | 프로젝트별 소스코드 (각자 별도 repo, gitignored) |
| `handoff/` | 워크스페이스 인덱스 (각 프로젝트 handoff 위치 안내) |
| `templates/` | 공통 양식 |
| `outputs/` | 워크스페이스 레벨 결과물 |

---

## 공유 컨텍스트 파일

| 파일 | 내용 | 갱신 조건 |
|------|------|----------|
| `context/about-me.md` | 사용자 역할·기술 스택·답변 선호 | 역할·기술 스택 변경 시 |
| `context/target-servers.md` | DeepGadget 제품군 사양 (모든 프로젝트가 dg5W/dg5R 등 참조) | 신규 제품군 추가 또는 단종 변경 시 |

프로젝트 전용 컨텍스트(검수 시스템 인프라, NIC 환경, 시스템 용어 등)는 각 프로젝트 내부 `context/`에 둔다.

`context/` 갱신 주체: 사용자 또는 사용자 지시를 받은 에이전트. 에이전트 독단 갱신 금지.

---

## 공통 규칙 파일

| 파일 | 내용 |
|------|------|
| `rules/api.md` | 코드 작성 공통 원칙 (보안·의존성·에러·구조) |
| `rules/writing.md` | 커밋·브랜치·문서 작성 규칙 |

---

## 글로벌 규칙

### CLI
- 모든 명령은 `rtk <cmd>` — 훅이 자동 변환

### Git / PR

| 구분 | 브랜치 정책 |
|------|-----------|
| **개인 solo repo** (예: 이 워크스페이스 자체) | `main` 직접 push 허용 |
| **팀 공유 repo** (예: inspection-system, p2p-perf-monitor) | `main` 직접 push 금지. `feature/`·`fix/`·`chore/` 브랜치 필수 |
| **외부 repo** (예: gadgetron) | 해당 repo의 기여 가이드라인 우선 |

각 프로젝트의 완료 기준(lint·test 명령)은 그 프로젝트의 `CLAUDE.md` "완료 워크플로우" 섹션 참조.

### 규칙 우선순위

충돌 시 아래 순서대로 적용 (높을수록 우선):

```
1. 프로젝트 .claude/rules/*.md   ← 가장 구체적, 항상 우선
2. 프로젝트 CLAUDE.md            ← 프로젝트 아키텍처·명령어
3. 워크스페이스 rules/*.md       ← 프로젝트 규칙 없는 영역에만 적용
4. 글로벌 ~/.claude/CLAUDE.md    ← 기본값
```

**판단 기준**: 같은 주제에 대해 여러 계층에 규칙이 있으면 가장 구체적인 계층(숫자 낮은 쪽)을 따름. 없는 영역은 상위 계층으로 fallback.

**예외**: 보안 규칙은 계층 무관 항상 적용.
