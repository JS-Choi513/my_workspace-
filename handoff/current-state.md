# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-08 (14차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 작업**: GPU stress 자동 설치 구현 (설계 확정, 미구현)

**전제조건 확인**:
```bash
git checkout main && git pull
pytest tests/ -x -q   # 263 passed, 8 skipped 기준
```

**주의**: 14차 세션 변경사항(SSH 수정 + preflight 재구성)이 아직 미커밋 상태.
구현 시작 전 이 변경사항 먼저 커밋/PR 처리 필요.

---

## 열린 PR 목록

| PR | 브랜치 | 내용 |
|----|--------|------|
| #33 | fix/cross-validation-harness | harness 교차검증 + 리뷰 반영 완료 — merge 대기 |

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
| **SSH auth 수정 + preflight sys-config 재구성** | — | ☐ 미커밋 |
| **GPU stress 자동 설치** | — | ☐ 미구현 |
| **WebGUI 프론트엔드** | — | ☐ 보류 |

---

## 14차 세션 완료 항목

### 실서버 테스트 (10.100.1.23) 중 발견 + 수정

**버그 1 — SSH 인증 실패 (미커밋)**
- 원인: `_build_connect_kwargs`에서 `_ssh_key_path()`가 미사용 키 파일(`/etc/inspection/ssh_keys/default`)을 발견해 항상 key auth 시도 → password auth 도달 불가
- 수정: password 우선, key는 fallback으로 순서 변경
- 파일: `workers/inspect.py`, `workers/sw_install.py` (두 파일 동일 패턴 수정)

**버그 2 — sleep.target FAIL (미커밋)**
- 원인: `_disable_auto_update`에 `sleep.target mask`가 없었음
- 수정: `systemctl mask sleep.target 2>&1 || true` 추가
- 파일: `workers/sw_install.py:_disable_auto_update`

**기능 개선 — Preflight sys-config 항상 적용 (미커밋)**
- 배경: sw_requirements 없으면 SW Install 스킵 → GRUB 파라미터 미적용됐음
- 수정: `_async_preflight`를 이중 세션 구조로 재구성
  - 세션 1: baseline + sys-config(GRUB 포함) → GRUB 변경 시 재부팅 후 세션 2
  - 세션 2: 재부팅 후 스크립트 실행
- 파일: `workers/inspect.py:_async_preflight`

**실서버 검증 결과**: `pass` (sw_auto_update, sw_power_mgmt 모두 정상 확인)

---

### GPU stress 자동 설치 — 설계 확정 (미구현)

**시나리오**: sw_requirements 없이 잡 제출 → GPU burn + CPU stress 자동 실행

**설계 결정**:
| 항목 | 결정 |
|------|------|
| driver 버전 | `580` (gpu_server.json `stress_config.driver_version`) |
| CUDA 버전 | `13` (gpu_server.json `stress_config.cuda_version`) |
| 재부팅 정책 | GRUB 변경 + driver 설치를 **단일 재부팅**으로 통합 |
| GPU 없는 서버 | 전체 스킵 |
| 완료 후 처리 | Cleanup에서 `apt purge` (driver + CUDA 모두 제거) |

**구현 계획**:

1. `checks/profiles/gpu_server.json`
   - `stress_config: { driver_version: "580", cuda_version: "13" }` 추가

2. `workers/sw_install.py`
   - `_nfs_temp_packages_path(job_id)` — NFS 경로 헬퍼
   - `_save_temp_packages(job_id, packages)` — 임시 설치 목록 저장
   - `_load_temp_packages(job_id)` — 임시 설치 목록 로드
   - `_check_driver_installed(conn)` — nvidia-smi 응답으로 driver 확인
   - `_check_cuda_installed(conn)` — nvcc 실행 가능 여부 확인

3. `workers/inspect.py:_async_preflight` — 단일 재부팅으로 재구성
   - 세션 1: sys-config → GRUB 변경 여부 기록(reboot 아직 X) → GPU 있고 driver 없으면 임시 설치 → `reboot_needed`면 단일 재부팅
   - 세션 2: 재부팅 후 reconnect → preflight 스크립트 실행

4. `workers/inspect.py:_async_post_install` — CUDA 임시 설치 추가
   - stress_tools 설치 후: GPU 있고 CUDA 없으면 임시 CUDA 설치 → temp_packages 갱신

5. `workers/inspect.py:_async_cleanup` — 임시 패키지 제거
   - `_load_temp_packages(job_id)` → 있으면 `apt purge`

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

## 미커밋 변경사항 (커밋 전 처리 필요)

| 파일 | 변경 내용 |
|------|----------|
| `workers/inspect.py` | `_build_connect_kwargs` password 우선 + preflight 이중 세션 재구성 |
| `workers/sw_install.py` | `_build_connect_kwargs` password 우선 + sleep.target mask 추가 |

커밋 방향: `fix/preflight-ssh-sysconfg` 브랜치로 PR 생성 권장

---

## 세션 기록

- [5차 세션](session-archive/session-5.md) — 문서 교차검증 2라운드, PR #10~14 머지 완료, PR #16 대기
