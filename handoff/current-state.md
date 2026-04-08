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
| harness 교차검증 5건 수정 | #33 | ✅ |
| SSH auth 수정 + preflight sys-config 재구성 + sleep.target | #34 | ✅ |
| GPU stress 자동 설치 (driver/CUDA 임시 설치 + cleanup) | #34 | ✅ |
| gpu_burn/nccl-tests $HOME 빌드 + baseline 엄격 실패 | #35 | ✅ |
| **WebGUI 프론트엔드** | — | ☐ 보류 |

---

## 15차 세션 완료 항목

### PR #35 — stress 도구 빌드 경로 + baseline 엄격화

**실서버(10.100.1.23) 검수 결과 gpu_burn 실제 미실행 발견 → 수정**:

1. **stress_gpu.py 재작성** — gpu_burn 전용
   - 빌드 경로 `/opt/gpu-burn` → `~/gpu-burn` (sudo 불필요)
   - dcgmi/pytorch fallback 제거 (silent skip 방지)
   - `ensure_gpu_burn()`: 바이너리 → nvcc/git/make 점검 → clone → make, 단계별 실패 사유(stderr 마지막 5줄) 명시
   - 실행 옵션: `gpu_burn -d -tc <duration>` (FP64 + Tensor Core)
   - `Popen(..., stderr=PIPE, text=True)` — 리뷰 반영, str/bytes 혼용 TypeError 방지

2. **nccl_bandwidth.py 재작성** — nccl-tests 전용
   - 빌드 경로 `/opt/nccl-tests` → `~/nccl-tests`
   - pytorch fallback 제거
   - 측정 실패 시 `warn` → `fail` 승격 + 사유

3. **baseline 설치 엄격화** (`workers/inspect.py`)
   - `_verify_packages()`, `_install_and_verify_baseline()` 추가
   - `dpkg -s` 로 패키지별 설치 여부 개별 검증
   - baseline 5개 중 1개라도 누락 → 즉시 `job=failed` + `_dispatch_cleanup`
   - `check_results` 에 `baseline_install` 항목으로 `installed/missing/apt_tail` 기록
   - `sudo_password` 미제공 케이스도 동일 처리 (warning → error 승격)
   - 이전: silent warning → 후속 단계가 nvme-cli 부재만 잡음, 근본 원인 손실
   - 변경: 어떤 패키지가 왜 실패했는지 DB·로그 모두 추적

4. **sw_storage_sw.py** — nvme 장치 + nvme-cli 미설치 시 안전망 `fail` (baseline 가드 우회 케이스 대비)

5. **cleanup 경로** — `gpu_server.json remove_dirs`: `$HOME/gpu-burn`, `$HOME/nccl-tests`. `inspect.py` cleanup이 `$HOME/`·`~/` prefix는 sudo 없이 `rm -rf`

**리뷰 반영**: stress_gpu Popen `text=True` 누락 (claude bot 지적, fda7d63)

**테스트**: 273 passed, 8 skipped

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
