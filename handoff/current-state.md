# Handoff — Inspection System v2 리팩토링

> 최종 업데이트: 2026-04-03 (3차 세션)  
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
| #10 | docs/sw-install-rules | 머지 대기 |
| #11 | docs/error-handling-rules | 머지 대기 |
| #12 | docs/migration-rules | 머지 대기 (리뷰 반영 완료: worker_sw_install 제거) |
| #13 | docs/sys-config-rules | 머지 대기 |

### 이번 세션에서 완료한 것

1. `migration.md` 작성 (PR #12) — Alembic 정책: 자동롤백, staging없음, 중단허용, 배포전 적용
2. `infra-environment.md` 작성 — cpu 서버(10.100.1.10/108), 동일 LAN, NFS 서버 겸임
3. `sys-config.md` 작성 (PR #13) — GRUB/CPU거버너/GPU PM/자동업데이트 방지, Ubuntu+Rocky
4. **전체 코드베이스 교차검증 완료** — 5개 Task 병렬 실행, 최종 리포트 작성

---

## 교차검증 결과 요약

### CRITICAL (구현 전 반드시 수정) — 7건

| # | 항목 | 위치 |
|---|------|------|
| C-1 | `sudo_password` SecretStr 미적용 — 평문 로그/메모리 노출 | `api/schemas.py:14`, `workers/inspect.py:78,225` |
| C-2 | WebSocket 인증 없음 — job_id만 알면 누구든 수신 | `api/websocket.py:32` |
| C-3 | `sw_requirements` 컬럼/필드 누락 | `api/models.py`, `api/schemas.py` |
| C-4 | `sw_gpu.py`/`sw_storage.py` 분리 미완 — preflight에 드라이버 의존 코드 혼재 | `checks/base/phase2_sw_basic/` |
| C-5 | `workers/app.py` include에 sw_install 누락 + celeryconfig q_sw_install 미정의 | `workers/app.py:5`, `config/celeryconfig.py` |
| C-6 | `tests/test_api/test_jobs.py` 부재 — 핵심 API 무테스트 | `tests/test_api/` |
| C-7 | `deploy.sh` 마이그레이션 단계 없음 | `scripts/deploy.sh` |

### WARNING — 10건 (구현 중 고려)

| # | 항목 |
|---|------|
| W-1 | JobStatus ENUM v2 미확장 (preflight, sw_install, rebooting 등 8개 누락) |
| W-2 | 재시도 간격 60s → 20s로 수정 필요 (`workers/inspect.py:335`) |
| W-3 | `known_hosts=None` → 명시적 경로 지정 권장 |
| W-4 | Redis 인증 없음 |
| W-5 | API 엔드포인트 인증 없음 |
| W-6 | 로그 볼륨 경로 불일치 (규칙: `/app/logs`, 실제: `/srv/inspection/logs`) |
| W-7 | `CheckResultResponse.raw_output` 누락 |
| W-8 | SSH mock 인터페이스 불일치 가능성 |
| W-9 | `sw_os_version.py` 잘못된 위치 (phase6 → preflight) |
| W-10 | `CheckResult` ENUM `create_type=False` 누락 |

### INFO — 8건 (구현 후 리팩토링)

config/logging.py 미구현, 프롬프트 인젝션, 파일 경로 검증, daily_check.sh 경로 하드코딩, 온도 임계값 중복, coverage 설정 없음, task_acks_late 중복, IB 상태 체크 불완전

### 교차검증 충돌

| 항목 | 결론 |
|------|------|
| `known_hosts=None` | Task1(정상) vs Task5(WARNING) → Task5 채택, 명시적 경로 지정 권장 |

---

## 다음 세션에서 시작할 작업

**추천 시작점:** CRITICAL 항목 처리 (C-1, C-2부터 — 독립적으로 처리 가능)

### CRITICAL 처리 순서 (권장)

1. **C-1 + C-3 묶음** (`api/schemas.py`, `api/models.py`)
   - `sudo_password: str` → `SecretStr`
   - `sw_requirements: str | None` 필드 추가
   - Alembic 마이그레이션 (v2: sw_requirements + JobStatus 확장)

2. **C-4** (`checks/base/` 재구성)
   - `phase2~7` → `preflight/` + `post_install/` + `collect/`
   - `sw_gpu` → `sw_gpu_hw.py` + `sw_gpu_sw.py`
   - `sw_storage` → `sw_storage_hw.py` + `sw_storage_sw.py`

3. **C-5** (`workers/app.py`, `config/celeryconfig.py`)
   - sw_install include 추가, q_sw_install 라우팅 추가

4. **C-6** (`tests/test_api/test_jobs.py` 신규 작성)

5. **C-7** (`scripts/deploy.sh` 마이그레이션 단계 삽입)

6. **C-2** (`api/websocket.py` 인증 추가)

---

## v2에서 바꿔야 하는 것 (전체 목록, 우선순위 순)

**[블로커] 아직 시작 안 된 항목:**

1. **`checks/base/` 재구성** — C-4 해결
2. **`workers/rule_validator.py` 신규** — `validation.rules` 기반 threshold 판정, 토큰 0
3. **`workers/agent_gateway.py` 신규** — 에이전트 호출 판단 + compact input 구성
4. **`workers/validate.py` 교체** — rule validator 우선, 에이전트 fallback
5. **`workers/ssh_client.py` 교체** — SecretStr 지원, pw 접속 후 즉시 폐기
6. **`workers/sw_planner.py` + `workers/sw_install.py` 신규** — SW 설치 파이프라인
7. **`workers/inspect.py` 교체** — preflight/post-install 단계 분리 + cleanup task
8. **`workers/app.py` 수정** — `q_sw_install` 큐 추가 — C-5 해결
9. **`api/schemas.py` 수정** — `sw_requirements` 필드 추가 + SecretStr — C-1, C-3 해결
10. **`api/models.py` 수정** — `Job.sw_requirements` Text 컬럼 + JobStatus 상태 확장
11. **`config/logging.py` 신규** — structlog 민감필드 마스킹
12. **`config/sw_compat_matrix.json` 신규** — driver/cuda/torch 버전 호환 lookup table
13. **Alembic 마이그레이션** — sw_requirements + JobStatus 상태 추가
14. **WebGUI 프론트엔드** — 미착수, 낮은 우선순위

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
