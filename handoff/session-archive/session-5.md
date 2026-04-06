# 5차 세션 아카이브 — 2026-04-05

## 완료 작업

**문서 교차검증 2라운드 + 개선 적용 (코드 변경 없음)**

1. **README.md v2 전면 재작성** — phase2~7 → preflight/post_install/collect, JobStatus 12개, Agent Zone 임계값, SSH 키 섹션 제거, `claude-sonnet-4-6` 현행화 (PR #16)
2. **CLAUDE.md 정리** — `(미작성...)` broken 참조 제거, `install_policy` 잔존 문구 삭제
3. **`.claude/rules/agents.md`** — 모델명 `claude-sonnet-4-6` 현행화, SW Planner `max_tokens` 1024→2048
4. **`.claude/rules/error-handling.md`** — deprecated `inspecting`/`error` 상태 명시, 로그 경로 `/app/logs` → `/srv/inspection/logs` (W-6 해소)
5. **`context/infra-environment.md`** — `CLAUDE_MODEL` 현행화
6. PR #16 머지 대기 중

## PR 상태

| PR | 브랜치 | 상태 |
|----|--------|------|
| #10 | docs/sw-install-rules | ✅ 머지 완료 |
| #11 | docs/error-handling-rules | ✅ 머지 완료 |
| #12 | docs/migration-rules | ✅ 머지 완료 |
| #13 | docs/sys-config-rules | ✅ 머지 완료 |
| #14 | fix/critical-c1-c3-c7 | ✅ 머지 완료 |
| #16 | chore/claude-fix-workflow-rtk | ⏳ 머지 대기 |

## 4차 세션 완료 작업

1. **CRITICAL C-1** `sudo_password` SecretStr 적용 — `api/schemas.py`, `api/routers/jobs.py`
2. **CRITICAL C-3** `sw_requirements` 필드/컬럼 추가 — `api/schemas.py`, `api/models.py`, `api/routers/jobs.py`
3. **CRITICAL C-3** `JobStatus` ENUM 8개 확장 + Alembic 마이그레이션 (`a1b2c3d4_...`)
4. **CRITICAL C-7** `deploy.sh` 마이그레이션 단계 삽입 (stop→build→migrate→restart 순서)
5. 연기된 항목(C-2/C-4/C-5/C-6) TODO 주석으로 코드에 마킹
6. PR #10~#14 리뷰 반영 및 전체 머지 완료

## CRITICAL 처리 현황

| # | 항목 | 상태 |
|---|------|------|
| C-1 | `sudo_password` SecretStr | ✅ 완료 — `inspect.py`는 ssh_client.py 리팩토링 시 통합 |
| C-2 | WebSocket 인증 | ⏸ 후순위 — W-5 API 인증과 묶음 (`api/websocket.py:35`) |
| C-3 | `sw_requirements` 필드/컬럼 | ✅ 완료 |
| C-4 | `checks/base/` 재구성 | ⏸ `inspect.py` 리팩토링 시 통합 (`gpu_server.json`) |
| C-5 | `workers/app.py` sw_install 큐 | ⏸ `sw_install.py` 구현 시 통합 (`workers/app.py:4`) |
| C-6 | `test_jobs.py` 부재 | ⏸ `inspect.py` 리팩토링 시 통합 (`tests/test_api/__init__.py`) |
| C-7 | `deploy.sh` 마이그레이션 누락 | ✅ 완료 |

## WARNING 10건

| # | 항목 | 상태 |
|---|------|------|
| W-1 | JobStatus ENUM v2 미확장 | ✅ C-3과 함께 완료 |
| W-2 | 재시도 간격 60s → 20s (`workers/inspect.py:335`) | 미처리 |
| W-3 | `known_hosts=None` → 명시적 경로 권장 | 미처리 |
| W-4 | Redis 인증 없음 | 미처리 |
| W-5 | API 엔드포인트 인증 없음 | 미처리 (C-2와 묶음) |
| W-6 | 로그 볼륨 경로 불일치 | ✅ 완료 — `/srv/inspection/logs`로 통일 |
| W-7 | `CheckResultResponse.raw_output` 누락 | 미처리 |
| W-8 | SSH mock 인터페이스 불일치 가능성 | 미처리 |
| W-9 | `sw_os_version.py` 잘못된 위치 | ⏸ C-4와 함께 처리 |
| W-10 | `CheckResult` ENUM `create_type=False` 누락 | 미처리 |

## INFO 8건 (구현 후 리팩토링)

config/logging.py 미구현, 프롬프트 인젝션, 파일 경로 검증, daily_check.sh 경로 하드코딩, 온도 임계값 중복, coverage 설정 없음, task_acks_late 중복, IB 상태 체크 불완전

## 교차검증 충돌

| 항목 | 결론 |
|------|------|
| `known_hosts=None` | Task1(정상) vs Task5(WARNING) → Task5 채택, 명시적 경로 지정 권장 |
