# Handoff — Inspection System v2 리팩토링

> 최종 업데이트: 2026-04-05 (5차 세션)  
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

### 완료된 문서 작업

| 문서 | 위치 | PR |
|------|------|----|
| `error-handling.md` | `.claude/rules/` | #11 머지 대기 |
| `sw-install.md` | `.claude/rules/` | #10 머지 대기 |
| `migration.md` | `.claude/rules/` | #12 머지 대기 |
| `sys-config.md` | `.claude/rules/` | #13 머지 대기 |
| `infra-environment.md` | `context/` (workspace) | auto-commit 반영 |
| `target-servers.md` | `context/` (workspace) | auto-commit 반영 |

### PR 상태 요약

| PR | 브랜치 | 상태 |
|----|--------|------|
| #10 | docs/sw-install-rules | ✅ 머지 완료 |
| #11 | docs/error-handling-rules | ✅ 머지 완료 |
| #12 | docs/migration-rules | ✅ 머지 완료 |
| #13 | docs/sys-config-rules | ✅ 머지 완료 |
| #14 | fix/critical-c1-c3-c7 | ✅ 머지 완료 |

### 이번 세션에서 완료한 것 (4차)

1. **CRITICAL C-1** `sudo_password` SecretStr 적용 — `api/schemas.py`, `api/routers/jobs.py`
2. **CRITICAL C-3** `sw_requirements` 필드/컬럼 추가 — `api/schemas.py`, `api/models.py`, `api/routers/jobs.py`
3. **CRITICAL C-3** `JobStatus` ENUM 8개 확장 + Alembic 마이그레이션 (`a1b2c3d4_...`)
4. **CRITICAL C-7** `deploy.sh` 마이그레이션 단계 삽입 (stop→build→migrate→restart 순서)
5. 연기된 항목(C-2/C-4/C-5/C-6) TODO 주석으로 코드에 마킹
6. PR #10~#14 리뷰 반영 및 전체 머지 완료

---

## 교차검증 결과 요약

### CRITICAL 처리 현황

| # | 항목 | 상태 |
|---|------|------|
| C-1 | `sudo_password` SecretStr | ✅ 완료 — `inspect.py`는 ssh_client.py 리팩토링 시 통합 (TODO 주석) |
| C-2 | WebSocket 인증 | ⏸ 후순위 — W-5 API 인증과 묶음 (TODO 주석: `api/websocket.py:35`) |
| C-3 | `sw_requirements` 필드/컬럼 | ✅ 완료 |
| C-4 | `checks/base/` 재구성 | ⏸ `inspect.py` 리팩토링 시 통합 (TODO 주석: `gpu_server.json`) |
| C-5 | `workers/app.py` sw_install 큐 | ⏸ `sw_install.py` 구현 시 통합 (TODO 주석: `workers/app.py:4`) |
| C-6 | `test_jobs.py` 부재 | ⏸ `inspect.py` 리팩토링 시 통합 (TODO 주석: `tests/test_api/__init__.py`) |
| C-7 | `deploy.sh` 마이그레이션 누락 | ✅ 완료 |

### WARNING — 10건 (구현 중 고려)

| # | 항목 | 상태 |
|---|------|------|
| W-1 | JobStatus ENUM v2 미확장 | ✅ C-3과 함께 완료 |
| W-2 | 재시도 간격 60s → 20s (`workers/inspect.py:335`) | 미처리 |
| W-3 | `known_hosts=None` → 명시적 경로 권장 | 미처리 |
| W-4 | Redis 인증 없음 | 미처리 |
| W-5 | API 엔드포인트 인증 없음 | 미처리 (C-2와 묶음) |
| W-6 | 로그 볼륨 경로 불일치 (`/app/logs` vs `/srv/inspection/logs`) | ✅ 완료 — `/srv/inspection/logs`로 통일 (error-handling.md) |
| W-7 | `CheckResultResponse.raw_output` 누락 | 미처리 |
| W-8 | SSH mock 인터페이스 불일치 가능성 | 미처리 |
| W-9 | `sw_os_version.py` 잘못된 위치 (phase6 → preflight) | ⏸ C-4와 함께 처리 |
| W-10 | `CheckResult` ENUM `create_type=False` 누락 | 미처리 |

### INFO — 8건 (구현 후 리팩토링)

config/logging.py 미구현, 프롬프트 인젝션, 파일 경로 검증, daily_check.sh 경로 하드코딩, 온도 임계값 중복, coverage 설정 없음, task_acks_late 중복, IB 상태 체크 불완전

### 교차검증 충돌

| 항목 | 결론 |
|------|------|
| `known_hosts=None` | Task1(정상) vs Task5(WARNING) → Task5 채택, 명시적 경로 지정 권장 |

---

## 다음 세션에서 시작할 작업

**추천 시작점:** `checks/base/` 재구성 + `workers/inspect.py` 리팩토링 (C-4와 묶음)

### TODO 주석 위치 (grep: `# TODO(C-`)

| 항목 | 파일 | 통합 시점 |
|------|------|----------|
| C-1 | `workers/inspect.py:78`, `:225` | `ssh_client.py` 리팩토링 시 |
| C-2 | `api/websocket.py:35` | W-5 API 인증 구현 시 |
| C-4 | `checks/profiles/gpu_server.json` | `inspect.py` 리팩토링 시 |
| C-5 | `workers/app.py:4`, `config/celeryconfig.py:9` | `sw_install.py` 구현 시 |
| C-6 | `tests/test_api/__init__.py:1` | `inspect.py` 리팩토링 시 |

---

## v2에서 바꿔야 하는 것 (전체 목록, 우선순위 순)

**[완료]**
- ✅ `api/schemas.py` — `sw_requirements` + `SecretStr` (C-1, C-3)
- ✅ `api/models.py` — `Job.sw_requirements` + `JobStatus` 확장 (C-3, W-1)
- ✅ `api/routers/jobs.py` — SecretStr `.get_secret_value()`, `sw_requirements` 저장
- ✅ `scripts/deploy.sh` — stop→build→migrate→restart 순서 (C-7)
- ✅ Alembic 마이그레이션 `a1b2c3d4` — `sw_requirements` + `JobStatus` 확장 (C-3)
- ✅ `.claude/rules/` 4개 문서 (error-handling, sw-install, migration, sys-config)

**[블로커] 아직 시작 안 된 항목:**

1. **`checks/base/` 재구성** — C-4: phase→preflight/post_install/collect, sw_gpu/sw_storage 분리 (`lspci` 기반 hw 로직 신규 작성)
2. **`workers/inspect.py` 교체** — preflight/post-install 단계 분리 + cleanup task (C-4, C-6과 묶음)
3. **`workers/ssh_client.py` 교체** — SecretStr 지원, pw 접속 후 즉시 폐기 (C-1 잔여 처리)
4. **`workers/rule_validator.py` 신규** — `validation.rules` 기반 threshold 판정, 토큰 0
5. **`workers/agent_gateway.py` 신규** — 에이전트 호출 판단 + compact input 구성
6. **`workers/validate.py` 교체** — rule validator 우선, 에이전트 fallback
7. **`workers/sw_planner.py` + `workers/sw_install.py` 신규** — SW 설치 파이프라인 (C-5와 묶음)
8. **`workers/app.py` 수정** — `q_sw_install` 큐 추가 (C-5)
9. **`config/logging.py` 신규** — structlog 민감필드 마스킹
10. **`config/sw_compat_matrix.json` 신규** — driver/cuda/torch 버전 호환 lookup table
11. **WebGUI 프론트엔드** — 미착수, 낮은 우선순위

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
| 재시도 간격 | error-handling.md 기준 20초 — inspect.py의 60초는 버그 (W-2) |
| known_hosts | 현재 None → 명시적 경로 `/etc/inspection/known_hosts` 지정 필요 (W-3) |
| 로그 볼륨 경로 | docker-compose: `/srv/inspection/logs`, 규칙 문서: `/app/logs` — 통일 필요 (W-6) |

---

## 인프라 환경 요약

| 항목 | 값 |
|------|-----|
| 검수 시스템 호스트 | `cpu` (10.100.1.10, 10.100.1.108) |
| NFS export | `/srv/inspection/results` → `10.100.1.0/24` |
| 검수 망 | `10.100.1.0/24` (대상 서버 동일 LAN) |
| SSH 인증 | 패스워드 방식 (키 인증 미사용) |

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
  .claude/rules/             ← error-handling, migration, sys-config, sw-install
  api/                       ← FastAPI
  workers/                   ← Celery tasks (v1: inspect, validate, report, notify)
  checks/base/               ← 검수 스크립트 (v2 재구성 필요: phase→preflight/post_install/collect)
  checks/profiles/           ← gpu_server.json
  config/                    ← settings, celeryconfig, prompts
  templates/                 ← Jinja2 리포트 템플릿
  alembic/                   ← DB 마이그레이션 (v1 스키마)
  tests/                     ← pytest (test_jobs.py 누락)
  scripts/                   ← deploy.sh (마이그레이션 단계 누락)
  context/ (workspace)       ← target-servers.md, infra-environment.md
```
