# API / 코드 작성 규칙

> 세부 규칙은 프로젝트별 `.claude/rules/api.md` 우선.
> 이 파일은 프로젝트 공통 원칙.

## Python 코드 원칙

- ruff lint/format (`line-length=100`)
- type hint 필수
- `str | None` 사용 (`Optional` 금지)
- f-string 사용 (`.format()` 금지)
- 구체 예외 catch, bare `except` 금지
- import 순서: stdlib → 3rd party → local
- API 레이어: async 함수 우선

## 보안

- 민감값(`password`, `token`, `api_key`)은 `SecretStr`로 선언
- 유저 입력을 shell 명령에 직접 삽입 금지
- password: DB 미저장, SSH 접속 후 즉시 폐기

## 의존성

- 새 패키지: `pyproject.toml` dependencies 반드시 반영
- 검수 스크립트(원격 실행): Python stdlib만 사용

