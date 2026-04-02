# 문서·커밋 작성 규칙

## 커밋 메시지

- 형식: `<type>(<scope>): <한 줄 요약>`
- type: `feat` / `fix` / `chore` / `refactor` / `test` / `docs`
- 예: `feat(workers): add sw_install task to q_sw_install`
- main 직접 push 금지

## 브랜치 명명

- `feature/<name>` — 신규 기능
- `fix/<name>` — 버그 수정
- `chore/<name>` — 설정·정리

## PR 조건

- `pytest tests/ -x -q` 통과
- `ruff check . && ruff format --check .` 통과
- 관련 `tests/` 포함

## 문서 파일 규칙

- 이 workspace의 `.md` 파일은 한국어 기본
- 코드 주석은 영어
- 테이블은 `|---|` 형식 (정렬 맞추지 않아도 됨)
- 코드 블록에 언어 명시 (` ```python `)

## handoff 문서 작성 원칙

- 다음 세션이 context 없이도 바로 작업 재개 가능해야 함
- "지금 어디까지 했다"보다 "다음에 뭘 해야 한다"를 중심으로
- 블로커(미해결 문제)와 결정 이유 명시
