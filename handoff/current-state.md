# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-05 (6차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**작업 블록**: `checks/base/` 재구성 + `workers/inspect.py` 리팩토링 (C-4, C-6 묶음)

**전제조건 확인**:
```bash
ls projects/inspection-system/checks/base/          # 현재 파일 목록
grep -r "# TODO(C-4" projects/inspection-system/     # TODO 위치 확인
git status                                           # 미커밋 변경사항 없음 확인
```

**실행 순서**:
1. `checks/base/` 하위에 `preflight/`, `post_install/`, `collect/` 디렉토리 생성
2. 기존 스크립트를 `inspection-scripts.md` Phase별 목록에 따라 이동
   - `sw_gpu.py` → `sw_gpu_hw.py`(preflight/) + `sw_gpu_sw.py`(post_install/) 분리
   - `sw_storage.py` → `sw_storage_hw.py`(preflight/) + `sw_storage_sw.py`(post_install/) 분리
   - 나머지 preflight 스크립트 → `preflight/`로 이동
   - stress/nccl 스크립트 → `post_install/`로 이동
   - `collect_all_logs.py` → `collect/`로 이동
3. `checks/profiles/gpu_server.json` `phases` 구조를 새 디렉토리 구조에 맞게 재작성
4. `workers/inspect.py`에서 preflight/post_install 단계 분리 + cleanup task 추가 (C-6: `tests/test_api/__init__.py:1` TODO 제거 포함)

**완료조건**:
- `pytest tests/ -x -q` 통과
- `python3 checks/base/preflight/sw_gpu_hw.py | python3 -m json.tool` 정상 출력
- `grep -r "# TODO(C-4\|# TODO(C-6" projects/inspection-system/` 결과 없음
- `ruff check . && ruff format --check .` 통과

---

## TODO 주석 위치 (`grep -r "# TODO(C-"`)

| 항목 | 파일:라인 | 통합 시점 |
|------|----------|----------|
| C-1 잔여 | `workers/inspect.py:78`, `:225` | `ssh_client.py` 리팩토링 시 |
| C-2 | `api/websocket.py:35` | W-5 API 인증 구현 시 |
| C-4 | `checks/profiles/gpu_server.json` | `inspect.py` 리팩토링 시 |
| C-5 | `workers/app.py:4`, `config/celeryconfig.py:9` | `sw_install.py` 구현 시 |
| C-6 | `tests/test_api/__init__.py:1` | `inspect.py` 리팩토링 시 |

---

## 미처리 블로커 (우선순위 순)

1. `checks/base/` 재구성 — C-4: phase→preflight/post_install/collect, sw_gpu/sw_storage 분리
2. `workers/inspect.py` 교체 — preflight/post-install 단계 분리 + cleanup task
3. `workers/ssh_client.py` 교체 — SecretStr 지원 (C-1 잔여)
4. `workers/rule_validator.py` 신규 — threshold 판정, 토큰 0
5. `workers/agent_gateway.py` 신규 — 에이전트 호출 판단 + compact input
6. `workers/validate.py` 교체 — rule validator 우선, 에이전트 fallback
7. `workers/sw_planner.py` + `workers/sw_install.py` 신규 (C-5)
8. `workers/app.py` — `q_sw_install` 큐 추가 (C-5)
9. `config/logging.py` 신규 — structlog 민감필드 마스킹
10. `config/sw_compat_matrix.json` 신규 — driver/cuda/torch 호환 lookup table

---

## 미처리 WARNING

W-2 재시도 간격 `inspect.py:335` 60s→20s / W-3 `known_hosts=None`→명시 경로 / W-4 Redis 인증 / W-5 API 인증 / W-7 `raw_output` 누락 / W-8 SSH mock / W-10 `create_type=False`

---

## 세션 기록

- [5차 세션](session-archive/session-5.md) — 문서 교차검증 2라운드, PR #10~14 머지 완료, PR #16 대기
