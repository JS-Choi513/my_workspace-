# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-08 (11차 세션)
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

없음 (전체 머지 완료)

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
| **WebGUI 프론트엔드** | — | ☐ 보류 |

---

## 11차 세션 완료 항목

### PR #31 — Blocker 9 (config/logging.py)

- `config/logging.py` 신규: `mask_sensitive_fields` structlog 프로세서
  - 키 완전 마스킹: `password`, `sudo_password`, `api_key`, `anthropic_api_key`, `token`, `secret` (대소문자 무관)
  - 문자열 내 `KEY=value` 패턴 치환 (traceback 환경변수 대응)
  - 중첩 dict / 리스트 재귀 처리
- `api/main.py`, `workers/app.py` 시작 시 `configure_logging()` 호출
- `tests/test_config/test_logging.py` 29개 테스트

### PR #32 — PR#31 리뷰 4건 수정

1. **`_handle_account` 쉘 인젝션**: `shlex.quote(username)` + 패스워드 → `sudo -S chpasswd` stdin 전달
2. **`_install_python` 3파트 버전**: `"3.11.5"` → `python3.11.5` 버그 → `[:2]` join으로 `python3.11` 생성
3. **retry 전 `_mark_failed` 제거**: `self.request.retries >= self.max_retries` 일 때만 호출 (`sw_install.py` + `inspect.py` 3곳)
4. **`DisconnectError` 억제 가드**: `"nvidia_driver" not in failed_deps` 조건 추가 — reboot trigger와 동일하게 맞춤

---

## TODO 주석 위치

| 항목 | 파일:라인 | 통합 시점 |
|------|----------|----------|
| W-3 | `workers/inspect.py:127` | known_hosts 명시 경로 |
| W-3 | `workers/sw_install.py:109` | known_hosts 명시 경로 |
| C-2 | `api/websocket.py:35` | W-5 API 인증 구현 시 |

---

## 미처리 WARNING

W-3 `known_hosts=None`→명시 경로 / W-4 Redis 인증 / W-5 API 인증 / W-7 `raw_output` 누락 / W-8 SSH mock / W-10 `create_type=False`

---

## 세션 기록

- [5차 세션](session-archive/session-5.md) — 문서 교차검증 2라운드, PR #10~14 머지 완료, PR #16 대기
