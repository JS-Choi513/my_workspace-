# Workspace

## 폴더 구조

| 폴더 | 역할 |
|------|------|
| `context/` | 자주 다시 설명하고 싶지 않은 규칙과 배경 (늘 펼쳐두는 참고철) |
| `rules/` | 코드·문서 작성 규칙 (프로젝트 공통 준수사항) |
| `projects/` | 프로젝트별 소스코드·원자료 (과제별 서류철) |
| `templates/` | 출력 구조 고정용 빈 양식 (양식 서랍) |
| `outputs/` | 사람이 다시 볼 최종 결과물 (결재함) |
| `handoff/` | 긴 작업을 다음 세션이 이어받는 메모함 |

## 현재 프로젝트

- `projects/inspection-system/` — GPU 서버 출고 전 검수 자동화 시스템

## 컨텍스트 파일 위치

- 배경·용어: `context/`
- 코딩·문서 규칙: `rules/`
- 세션 인수인계: `handoff/current-state.md`

## 글로벌 규칙

- RTK 사용: 모든 CLI 명령은 `rtk <cmd>` 형식으로
- 프로젝트별 규칙은 해당 프로젝트 `CLAUDE.md` 및 `.claude/rules/` 우선
