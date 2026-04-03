# Handoff — Inspection System v2 리팩토링

> 최종 업데이트: 2026-04-03  
> 다음 세션이 이 파일만 읽고 작업을 바로 재개할 수 있도록 작성.

---

## 한 줄 요약

GPU 서버 출고 전 검수 자동화 시스템. **v1 완료, v2 리팩토링 진행 중.**  
v2의 핵심 변경: "LLM은 판단에만, 실행은 코드가" 원칙 강화 — rule_validator로 토큰 0 정상 플로우 구현.

---

## 프로젝트 위치

```
~/workspace/projects/inspection-system/
```

---

## 현재 상태

### v1 완료된 것

- Docker Compose 전체 스택 (api, worker_inspect, worker_validate, worker_report, redis, postgres, flower)
- DB 모델 (Job, CheckResult, Report) + Alembic 마이그레이션
- Jobs API (POST/GET/DELETE) + Reports API + WebSocket
- Celery 큐 3개: `q_inspect`, `q_validate`, `q_report`
- 검수 스크립트 12개 (`checks/base/`) — phase2~7 구조 (v2에서 재구성 예정)
- Validate Worker: Claude API 전체 결과 전달 방식 (v2에서 교체 예정)
- Report Worker: PDF/XLSX 생성 (Jinja2 + WeasyPrint + openpyxl)

### 이번 세션에서 완료한 것 (문서/rules)

- README.md v2 아키텍처 전면 재작성 (PR #6 머지 완료)
- README.md Mermaid 다이어그램으로 교체, Agent Layer 가시화 (PR #9 머지 완료)
- `.claude/rules/sw-install.md` 신규 작성 (PR #10, 머지 대기)
  - 트리거 조건, sw_requirements 파싱 규칙, 항목 분류
  - 버전 호환 lookup table 구조 (`config/sw_compat_matrix.json`)
  - 설치 순서·의존성 전체 정의
  - 항목별 설치 절차 및 최소 검증 (nvidia-driver, CUDA, torch, docker, miniconda, tt-kmd 등)
  - reboot 처리 패턴 (300s SSH 재접속 폴링)
  - 계정 생성, 스토리지 마운트, 실패 처리
- `install_policy` 필드 제거 결정 — sw_requirements 유무로만 분기
- `.gitignore` 수정 — `.claude/rules/`, `.claude/commands/`, `.claude/skills/` 추적 허용
- `.claude/rules/error-handling.md` 신규 작성 (미커밋)
  - 파이프라인 실패 전파 규칙 (Cleanup+Report 항상 실행, SW Install 실패 시 Post-install 건너뜀)
  - JobStatus 전체 목록 (REJECTED 상태 포함)
  - 재시도 정책 (SSH/API 실패 3회·20초, 그 외 즉시 FAILED)
  - SSH 연결 실패 vs 스크립트 실행 실패 구분 및 error_type 정의
  - 에러 로그 경로·포맷 (JSONL, task별 분리)
  - 민감정보 마스킹 규칙 (structlog processor)

### v2에서 바꿔야 하는 것 (우선순위 순)

**[블로커] 아직 시작 안 된 항목:**

1. **`checks/base/` 재구성** — 현재 `phase2_sw_basic/` 등 구식 구조
   - `preflight/` + `post_install/` + `collect/` 디렉토리로 이동
   - `sw_gpu` → `sw_gpu_hw.py` (preflight) + `sw_gpu_sw.py` (post_install) 분리
   - `sw_storage` → `sw_storage_hw.py` + `sw_storage_sw.py` 분리

2. **`workers/rule_validator.py` 신규** — `validation.rules` 기반 threshold 판정, 토큰 0
3. **`workers/agent_gateway.py` 신규** — 에이전트 호출 판단 + compact input 구성
4. **`workers/validate.py` 교체** — rule validator 우선, 에이전트 fallback
5. **`workers/ssh_client.py` 교체** — SecretStr 지원, pw 접속 후 즉시 폐기
6. **`workers/sw_planner.py` + `workers/sw_install.py` 신규** — SW 설치 파이프라인
7. **`workers/inspect.py` 교체** — preflight/post-install 단계 분리 + cleanup task
8. **`workers/app.py` 수정** — `q_sw_install` 큐 추가
9. **`api/schemas.py` 수정** — `sw_requirements` 필드 추가 (install_policy 제거됨)
10. **`api/models.py` 수정** — `Job.sw_requirements` Text 컬럼 + JobStatus 상태 확장
11. **`config/logging.py` 신규** — structlog 민감필드 마스킹
12. **`config/sw_compat_matrix.json` 신규** — driver/cuda/torch 버전 호환 lookup table
13. **Alembic 마이그레이션** — sw_requirements + 신규 JobStatus 상태 추가
14. **WebGUI 프론트엔드** — 미착수, 낮은 우선순위

---

## 다음 세션에서 시작할 작업

**추천 시작점:** `rules/error-handling.md` 작성 (문서 작업 계속)

이유: 추가 문서 작업이 남아 있음. error-handling 규칙은 v2 Worker 구현(Phase 3~4) 전에 반드시 확정되어야 함.

### 남은 문서 작업 (우선순위 순)

| 순서 | 문서 | 상태 |
|------|------|------|
| 1 | `.claude/rules/error-handling.md` | 미작성, 다음 세션 시작점 |
| 2 | `context/target-servers.md` | 미작성 |
| 3 | `.claude/rules/migration.md` | 미작성 |
| 4 | `context/infra-environment.md` | 미작성 |
| 5 | 필수 시스템 설정 목록 문서 | 미작성 (E 항목에서 파생) |

### error-handling.md 작성을 위해 물어봐야 할 것

- Celery chain 중 한 단계 실패 시 후속 task 처리 방식 (취소? 계속?)
- `self.retry()` 사용 조건과 횟수 정책
- Agent 호출 후 복구 실패 시 job 전체 상태 처리
- SSH 연결 실패 vs 스크립트 실행 실패 구분 방식
- 에러 로그 저장 위치 및 포맷

---

## 아키텍처 v2 플로우

```
[User] → job 제출 (서버정보 + 기대스펙 + SW요구사항.md)
  ↓
[Preflight Runner]    q_inspect  — baseline 설치 → HW/OS 점검
  ↓ (에러 시만) → [Inspect Agent]
[SW Install Runner]   q_sw_install — SW 요구사항 기반 설치 (없으면 skip)
  ↓ (비정형/실패 시만) → [SW Planner Agent]
[Post-install Runner] q_inspect  — stress_tools 설치 → 본검수 + Stress
[Rule Validator]      q_validate — threshold 기반 PASS/FAIL (토큰 0)
  ↓ (경계값/복합WARN 시만) → [Verify Agent]
[Cleanup Runner]      q_inspect  — 검수 전용 도구 제거
[Report Generator]    q_report   — Jinja2 → PDF/XLSX
```

---

## 알려진 이슈 / 결정 사항

| 항목 | 내용 |
|------|------|
| Alembic ENUM | `postgresql.ENUM(..., create_type=False)` + `DO $$ EXCEPTION WHEN duplicate_object $$` 패턴 필수 |
| DB 초기화 | `alembic_version` 테이블도 함께 DROP 후 재마이그레이션 |
| password 처리 | DB 미저장, 로그 마스킹, SSH 접속 후 즉시 폐기. 재검사 시 유저 재입력 |
| stress timeout | soft `7200s`, hard `7500s` |
| `task_acks_late=True` | 워커 crash 시 재할당 보장 |
| shell=True 이유 | 검수 스크립트는 원격 서버 1회성 실행, injection 입력 없음 (환경변수만) |
| install_policy | 제거됨. sw_requirements 유무로만 SW Install 단계 분기 |
| reboot 처리 | nvidia-driver 설치 후 필수. 300s SSH 재접속 폴링, 동일 Celery task가 재접속 후 이어서 실행 |
| 검증 역할 분리 | SW Install Worker = "설치됐는가" (최소 검증), post_install scripts = "제대로 동작하는가" |
| Ubuntu 우선 | 검수 대상 서버는 Ubuntu 22.04/24.04가 절대 다수. Rocky/RHEL은 Agent 판단, 추후 스크립트 추가 |
| 계정 패스워드 | sw_requirements.md에 평문 기재 → SecretStr 처리, DB 미저장 |
| 기타 시스템 설정 | grub/crontab/hibernate 등은 스크립트 아닌 SW Planner Agent 처리 |

---

## 빠른 기동

```bash
cd ~/workspace/projects/inspection-system
docker compose up -d
docker compose exec api alembic upgrade head
# 확인: http://localhost:8000/docs, http://localhost:5555
```

---

## 주요 파일 지도

```
projects/inspection-system/
  CLAUDE.md                  ← 이 프로젝트의 Claude 지시
  .claude/rules/             ← 세부 규칙 (api, workers, agents, profiles, testing, database, security, sw-install)
  api/                       ← FastAPI
  workers/                   ← Celery tasks
  checks/base/               ← 검수 스크립트 (v2 재구성 필요)
  checks/profiles/           ← 프로파일 JSON
  config/                    ← settings, celeryconfig, prompts
  templates/                 ← Jinja2 리포트 템플릿
  alembic/                   ← DB 마이그레이션
  tests/                     ← pytest
```
