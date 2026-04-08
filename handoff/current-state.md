# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-08 (12차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 작업**: WebGUI 프론트엔드 (보류 중 — 기술 방향 미확정)

**전제조건 확인**:
```bash
git checkout main && git pull
pytest tests/ -x -q   # 263 passed, 8 skipped (main 기준)
```

---

## 열린 PR 목록

| PR | 브랜치 | 내용 |
|----|--------|------|
| #33 | fix/cross-validation-harness | harness 교차검증 5건 수정 — 머지 대기 |

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
| **WebGUI 프론트엔드** | — | ☐ 보류 |

---

## 12차 세션 완료 항목

### PR #33 — harness 교차검증 5건 수정

병렬 서브에이전트 4개로 harness rules 13개 vs 구현 코드 교차검증 수행.
15건 발견 → 직접 코드 확인 → 10건 false positive 기각, 5건 반영.

**반영 항목**:

1. **`workers/validate.py`**: `RateLimitError` / `APIConnectionError` 재시도 중 `_mark_failed` 조기 호출 제거 → `retries >= max_retries`일 때만 호출
2. **`workers/sw_install.py`**: GRUB 파라미터 변경 후 nvidia_driver 재부팅 없는 경우 `grub_only_reboot_triggered` 플래그로 별도 재부팅 처리
3. **`api/schemas.py` + `api/models.py`**: `hw_manual_checks: dict | None` 필드 누락 추가
4. **`alembic/versions/c3d4e5f6`**: `jobs.hw_manual_checks` JSON 컬럼 마이그레이션
5. **`config/settings.py` + `.env.example`**: 미사용 `claude_max_tokens` 제거
6. **`checks/profiles/gpu_server.json`**: baseline에 `smartctl` 추가

**기각 항목 (false positive)**:
- C-1: phase_env SUDO_PASSWORD — inspection-scripts.md 설계대로 (env var 전달 의도된 패턴)
- C-2: inspect.py APIConnectionError — inspect.py는 Anthropic API 직접 호출 없음
- C-3: 스크립트 실패 FAILED 미처리 — caller에서 정상 호출됨
- C-4: 비Ubuntu sys-config skip — 코드에 agent 경유 구현 확인
- M-3: sys_config CheckResult 미저장 — install_results 루프에서 전부 저장 확인

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
