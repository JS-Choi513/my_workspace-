# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-07 (9차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 블로커**: Blocker 7b — `workers/sw_install.py` 신규 + Blocker 8 `workers/app.py` q_sw_install 큐 추가

**전제조건 확인**:
```bash
git log --oneline -5
gh pr list --state all --limit 5   # PR #28 머지 여부 확인
git checkout main && git pull
```

**작업 순서 권장**:
1. PR #28 머지 확인 후 진행
2. Blocker 7b: `workers/sw_install.py` 신규 (q_sw_install 태스크)
3. Blocker 8: `workers/app.py` — q_sw_install 큐 추가 (sw_install.py와 같은 PR)
4. Blocker 9: `config/logging.py` — structlog 민감필드 마스킹

---

## 열린 PR 목록 (세션 종료 시점)

| PR | 브랜치 | 내용 |
|----|--------|------|
| #28 | `feature/sw-planner` | workers/sw_planner.py + sw_compat_matrix.json (Blocker 7a + 10) |

---

## TODO 주석 위치

| 항목 | 파일:라인 | 통합 시점 |
|------|----------|----------|
| W-3 | `workers/inspect.py:127` | known_hosts 명시 경로 |
| C-2 | `api/websocket.py:35` | W-5 API 인증 구현 시 |
| C-5 | `workers/app.py:5` | `sw_install.py` 구현 시 (Blocker 8) |
| C-5 | `config/celeryconfig.py:8` | `sw_install.py` 구현 시 (Blocker 8) |

---

## 미처리 블로커 (우선순위 순)

1. `workers/sw_install.py` 신규 + `workers/app.py` q_sw_install 큐 (Blocker 7b + 8) ← **다음 작업**
2. `config/logging.py` 신규 — structlog 민감필드 마스킹 (Blocker 9)

> Blocker 10 (`config/sw_compat_matrix.json`)은 9차 세션 PR #28에 포함되어 완료.

---

## 9차 세션 완료 항목

### PR #28 — Blocker 7a + Blocker 10

- **workers/sw_planner.py** 신규
  - `parse(md_text)`: 불릿 라인 → 4가지 분류 (sw_install / account / storage_mount / sys_config)
  - `build_plan(job_id, sw_requirements)`: 파싱 + 호환성 확인 + agent 에스컬레이션
  - `_check_compat()`: driver+cuda, cuda+torch 매트릭스 조회 → 불호환 시 agent_required=True
- **workers/agent_gateway.py**: `call_sw_planner_agent` stub → 실구현
  - failed_step 재계획 지원, plan 타입 정규화, 에러 시 graceful fallback
- **config/sw_compat_matrix.json**: driver/cuda/torch 호환 lookup table
- **config/prompts/sw_planner_agent.txt**: SW Planner Agent 프롬프트
- **tests/test_workers/test_sw_planner.py**: 30개 신규
- **tests/test_workers/test_agent_gateway.py**: stub 테스트 → sw_planner 실구현 테스트 교체

---

## 미처리 WARNING

W-3 `known_hosts=None`→명시 경로 / W-4 Redis 인증 / W-5 API 인증 / W-7 `raw_output` 누락 / W-8 SSH mock / W-10 `create_type=False`

---

## 세션 기록

- [5차 세션](session-archive/session-5.md) — 문서 교차검증 2라운드, PR #10~14 머지 완료, PR #16 대기
