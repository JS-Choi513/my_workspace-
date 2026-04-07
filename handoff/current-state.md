# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-06 (7차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 블로커**: `workers/rule_validator.py` 신규 (Blocker 4)

**전제조건 확인**:
```bash
git log --oneline -5          # 최근 커밋 확인
gh pr list                    # 열린 PR 확인 (PR #22, #23, #24 머지 여부)
grep -r "# TODO(C-" projects/inspection-system/  # 잔여 TODO 위치
```

**작업 순서 권장** (블로커 우선순위 순):
1. PR #22, #23, #24 머지 확인 후 진행 (현재 OPEN)
2. Blocker 4: `workers/rule_validator.py` — threshold 판정, 토큰 0
3. Blocker 5: `workers/agent_gateway.py` — 에이전트 호출 판단 + compact input
4. Blocker 6: `workers/validate.py` 교체 — rule validator 우선, 에이전트 fallback

---

## 열린 PR 목록 (세션 종료 시점)

| PR | 브랜치 | 내용 |
|----|--------|------|
| #22 | `feature/v2-inspect-phases` | checks/base/ 재구성 + inspect.py 3 task 분리 (C-4, C-6) |
| #23 | `feature/v2-ssh-client-secstr` | ssh_client.py SecretStr 래퍼 (C-1) |
| #24 | `fix/pr22-review-feedback` | PR#22 리뷰 피드백 4건 수정 |

---

## TODO 주석 위치 (`grep -r "# TODO(C-\|# TODO(W-"`)

| 항목 | 파일:라인 | 통합 시점 |
|------|----------|----------|
| W-3 | `workers/inspect.py:127` | known_hosts 명시 경로 |
| C-2 | `api/websocket.py:35` | W-5 API 인증 구현 시 |
| C-5 | `workers/app.py:5` | `sw_install.py` 구현 시 |
| C-5 | `config/celeryconfig.py:8` | `sw_install.py` 구현 시 |

---

## 미처리 블로커 (우선순위 순)

1. `workers/rule_validator.py` 신규 — threshold 판정, 토큰 0 ← **다음 작업**
2. `workers/agent_gateway.py` 신규 — 에이전트 호출 판단 + compact input
3. `workers/validate.py` 교체 — rule validator 우선, 에이전트 fallback
4. `workers/sw_planner.py` + `workers/sw_install.py` 신규 (C-5)
5. `workers/app.py` — `q_sw_install` 큐 추가 (C-5)
6. `config/logging.py` 신규 — structlog 민감필드 마스킹
7. `config/sw_compat_matrix.json` 신규 — driver/cuda/torch 호환 lookup table

---

## 7차 세션 완료 항목

### C-4 + C-6 (PR #22)
- `checks/base/` → `preflight/` + `post_install/` + `collect/` 재구성
- `sw_gpu` → `sw_gpu_hw.py`(preflight) + `sw_gpu_sw.py`(post_install) 분리
- `sw_storage` → `sw_storage_hw.py` + `sw_storage_sw.py` 분리
- `workers/inspect.py` → `run_preflight` + `run_post_install` + `run_collect` + `run_cleanup` 4 task
- `checks/profiles/gpu_server.json` v2 구조로 재작성
- 구 `phase*/` 디렉토리 삭제

### C-1 (PR #23)
- `workers/ssh_client.py` 신규: `wrap_password()`, `secret_input()`
- `inspect.py` 각 async 함수 진입 직후 `SecretStr` 변환
- `_apt_install` 파라미터 `str → SecretStr`

### PR #22 리뷰 피드백 (PR #24)
- **P1**: `report.py` — `claude_verdict.json` 없을 때 DB fallback verdict 합성
- **P1**: `sw_storage_sw.py` — SATA/SAS 디스크 `smartctl -H` 헬스 체크 추가
- **P2**: `run_cleanup` — 재시도 소진 후에만 `_mark_report_trigger` 호출
- **P2**: `run_collect` 신규 task — `post_install → collect → validate` 체인 구성

---

## 미처리 WARNING

W-2 재시도 간격 60s→20s (완료됨) / W-3 `known_hosts=None`→명시 경로 / W-4 Redis 인증 / W-5 API 인증 / W-7 `raw_output` 누락 / W-8 SSH mock / W-10 `create_type=False`

---

## 세션 기록

- [5차 세션](session-archive/session-5.md) — 문서 교차검증 2라운드, PR #10~14 머지 완료, PR #16 대기
