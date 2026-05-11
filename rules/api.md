# 코드 작성 공통 원칙

> 워크스페이스 공통 fallback. **프로젝트별 `.claude/rules/`가 우선**.
> 각 프로젝트가 사용하는 언어·도구에 따른 구체 규칙은 그 프로젝트 내부에서 정의.

## 보안

- 민감값(`password`, `token`, `api_key`, secret 일체)은 로그·응답·에러 메시지에 노출 금지
- 유저 입력을 shell 명령에 직접 삽입 금지 (인자 분리 / allowlist 검증)
- 비밀번호: DB 미저장, 사용 직후 메모리에서 폐기. 정적 분석에서 적발되도록 언어별 secret 타입 사용 (Python `SecretStr`, Rust `secrecy::SecretString` 등)

## 의존성

- 새 패키지: 해당 프로젝트의 매니페스트(`pyproject.toml` / `Cargo.toml` / `package.json` 등)에 명시
- 외부 CDN·postinstall 외부 다운로드 금지 (폐쇄망·재현성)

## 에러 처리

- bare catch-all 금지 (`except:` / `catch (...)` / `unwrap()` 남발). 구체 예외·에러 타입 명시
- 에러 메시지에 컨텍스트 포함 (어떤 단계, 어떤 입력에서 발생했는지)

## 구조

- import / use 순서: 표준 라이브러리 → 외부 의존성 → 로컬
- public API는 타입 시그니처 명시 (Python type hint, Rust 시그니처, TS 타입)
- I/O 경계는 async, 순수 계산은 sync (언어가 비동기 지원하는 경우)
