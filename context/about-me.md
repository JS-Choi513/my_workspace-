# About Me

## 역할

GPU 서버 출고 전 검수 자동화 시스템(inspection-system)을 개발하는 엔지니어.
FastAPI, Celery, Python 기반 백엔드 개발이 주 업무.

## 작업 방식 선호

- 코드 먼저 보여주고 설명은 짧게
- 완성된 파일 단위로 작업, 패치 조각 나열 지양
- 커밋 전 `pytest` + `ruff` 통과 필수
- main 직접 push 금지, `feature/` `fix/` `chore/` 브랜치 사용

## 도구

- RTK (`rtk <cmd>`): 토큰 절감용 CLI 프록시, 모든 명령에 prefix
- Claude Code CLI (이 환경)
- Docker Compose: 인프라 로컬 기동
- Flower(`:5555`): Celery 태스크 모니터링

## 핵심 철학

**"LLM은 판단에만, 실행은 코드가"**
— 정상 플로우에서 에이전트 호출 없음, 토큰 0이 목표
