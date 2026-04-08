# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-08 (15차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 작업**: WebGUI 프론트엔드 (보류 중 — 기술 방향 미확정)

**전제조건 확인**:
```bash
git checkout main && git pull
pytest tests/ -x -q   # 270 passed, 8 skipped
```

---

## 열린 PR 목록

(없음)

최근 머지: #33, #34, #35

---

## v2 구현 완료 현황

| 항목 | PR | 상태 |
|------|----|------|
| api/schemas.py, api/models.py, Alembic 마이그레이션 | — | ✅ |
| checks/base/ 재구성 (preflight/ post_install/ collect/) | — | ✅ |
| workers/inspect.py (preflight/post_install/collect/cleanup 4 태스크) | — | ✅ |
| workers/ssh_client.py (SecretStr + pw 폐기) | — | ✅ |
| workers/rule_validator.py | #25 | ✅ |
| workers/agent_gateway.py | #26 | ✅ |
| workers/validate.py (v2) | #27 | ✅ |
| workers/sw_planner.py | #28 | ✅ |
| workers/sw_install.py + q_sw_install 큐 | #30 | ✅ |
| config/logging.py (structlog 민감필드 마스킹) | #31 | ✅ |
| sw_install 보안·정확성 버그 4건 수정 | #32 | ✅ |
| harness 교차검증 5건 수정 | #33 | ☐ 머지 대기 |
| SSH auth 수정 + preflight sys-config 재구성 + sleep.target | #34 | ☐ 머지 대기 |
| GPU stress 자동 설치 (driver/CUDA 임시 설치 + cleanup) | #34 | ☐ 머지 대기 |
| **WebGUI 프론트엔드** | — | ☐ 보류 |

---

## 14차 세션 완료 항목

### PR #34 — 실서버 버그 수정 + GPU stress 자동 설치

**실서버 테스트(10.100.1.23) 중 발견 수정 3건**:

1. **SSH auth 실패**: `_build_connect_kwargs` — 미사용 key 파일이 발견돼 password auth 미도달
   → password 우선, key fallback 순서로 변경 (`inspect.py`, `sw_install.py`)

2. **sleep.target FAIL**: `_disable_auto_update`에 `systemctl mask sleep.target` 누락
   → 추가 (`sw_install.py`)

3. **preflight sys-config 미적용**: sw_requirements 없으면 SW Install 스킵 → GRUB 미적용
   → `_async_preflight` 이중 세션 재구성 (항상 sys-config 적용) (`inspect.py`)

**GPU stress 자동 설치 구현**:
- `gpu_server.json`: `stress_config: {driver_version: "580", cuda_version: "13"}` 추가
- `sw_install.py`: `_check_driver_installed`, `_check_cuda_installed`, `_save/_load_temp_packages` 5개 헬퍼
- `inspect.py:_async_preflight`: GRUB + driver 임시 설치 → **단일 재부팅** 통합
- `inspect.py:_async_post_install`: CUDA 미설치 시 임시 설치 (stress_gpu.py gpu_burn 전제)
- `inspect.py:_async_cleanup`: `temp_packages.json` 읽어 `apt purge` 제거

**부수 버그 수정** (test_validate.py):
- 세션 13 추가 테스트 2건 — `validate_results.run(mock_self, ...)` 잘못된 패턴
  (bound method라 `mock_self`가 `job_id`로 들어감)
  → `validate_results.run.__func__(mock_self, ...)` 로 수정
- `httpx.Response(429)` → `httpx.Response(429, request=request)` (request 누락 수정)

**테스트**: 270 passed, 8 skipped

---

## TODO 주석 위치

| 항목 | 파일:라인 | 통합 시점 |
|------|----------|----------|
| W-3 | `workers/inspect.py:127` | known_hosts 명시 경로 |
| W-3 | `workers/sw_install.py:149` | known_hosts 명시 경로 |
| C-2 | `api/websocket.py:35` | W-5 API 인증 구현 시 |

---

## 미처리 WARNING

W-3 `known_hosts=None`→명시 경로 / W-4 Redis 인증 / W-5 API 인증 / W-7 `raw_output` 누락 / W-8 SSH mock / W-10 `create_type=False`

---

## 세션 기록

- [5차 세션](session-archive/session-5.md) — 문서 교차검증 2라운드, PR #10~14 머지 완료, PR #16 대기
