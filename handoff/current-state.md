# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-07 (8차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 블로커**: Blocker 7 — `workers/sw_planner.py` + `workers/sw_install.py` 신규 (C-5)

**전제조건 확인**:
```bash
git log --oneline -5
gh pr list                   # PR #25, #26, #27 머지 여부 확인
git checkout main && git pull
```

**작업 순서 권장** (블로커 우선순위 순):
1. PR #25, #26, #27 머지 확인 후 진행 (현재 OPEN)
2. Blocker 7: `workers/sw_planner.py` + `workers/sw_install.py` (C-5)
3. Blocker 8: `workers/app.py` — q_sw_install 큐 추가
4. Blocker 9: `config/logging.py` — structlog 민감필드 마스킹
5. Blocker 10: `config/sw_compat_matrix.json` — driver/cuda/torch 호환 lookup table

---

## 열린 PR 목록 (세션 종료 시점)

| PR | 브랜치 | 내용 |
|----|--------|------|
| #25 | `feature/rule-validator` | workers/rule_validator.py 신규 (Blocker 4) |
| #26 | `feature/agent-gateway` | workers/agent_gateway.py 신규 (Blocker 5) |
| #27 | `feature/validate-v2` | workers/validate.py v2 교체 (Blocker 6) |

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

1. `workers/sw_planner.py` + `workers/sw_install.py` 신규 (C-5) ← **다음 작업**
2. `workers/app.py` — `q_sw_install` 큐 추가 (C-5)
3. `config/logging.py` 신규 — structlog 민감필드 마스킹
4. `config/sw_compat_matrix.json` 신규 — driver/cuda/torch 호환 lookup table

---

## 8차 세션 완료 항목

### PR #25 리뷰 피드백 반영 (rule_validator.py)
- **P1**: missing metric + check.status=fail → fail_closed 처리 (기존: 규칙 건너뜀)
- **P2**: detail_map 중복 check_results 시 created_at 최신 레코드 선택 (newest-wins)
- 신규 테스트 4개: missing_metric_on_failed_check, duplicate_dedup ×2

### PR #26 리뷰 피드백 반영 (agent_gateway.py)
- **P1**: `_parse_json_response` — non-dict JSON(list, str 등) 반환 시 None 처리
- 신규 테스트 2개: parse_json_list_returns_none, parse_json_string_returns_none

### Blocker 6 — workers/validate.py v2 교체 (PR #27)
- v1 (Claude API 직접 호출) → v2 (rule_validator 우선, Verify Agent fallback)
- 플로우: `rule_evaluate()` → pass/fail/agent_required 분기
  - pass → job.status=cleanup → cleanup 트리거
  - fail → job.status=failed → cleanup 트리거
  - agent_required → `call_verify_agent()` → pass/reject → cleanup 트리거
- `claude_verdict.json` v2 포맷: verdict/fail_items/warn_items/warn_count/agent_verdict
- cleanup은 판정 결과와 무관하게 항상 실행
- `expected_specs` DB 컬럼 추가: Job 모델 + JobCreate 스키마 + jobs 라우터 + Alembic 마이그레이션 `b2c3d4e5`
- 테스트 v2 전면 교체 (11개)

---

## 미처리 WARNING

W-3 `known_hosts=None`→명시 경로 / W-4 Redis 인증 / W-5 API 인증 / W-7 `raw_output` 누락 / W-8 SSH mock / W-10 `create_type=False`

---

## 세션 기록

- [5차 세션](session-archive/session-5.md) — 문서 교차검증 2라운드, PR #10~14 머지 완료, PR #16 대기
