# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-07 (10차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 블로커**: Blocker 9 — `config/logging.py` structlog 민감필드 마스킹

**전제조건 확인**:
```bash
gh pr list --state all --limit 5   # PR #30 머지 여부 확인
git checkout main && git pull
```

**작업 순서 권장**:
1. PR #30 머지 확인 후 진행
2. Blocker 9: `config/logging.py` 신규 — structlog 민감필드 마스킹

---

## 열린 PR 목록 (세션 종료 시점)

| PR | 브랜치 | 내용 | 상태 |
|----|--------|------|------|
| #30 | `feature/sw-install` | workers/sw_install.py 신규 + q_sw_install 큐 (Blocker 7b + 8) | OPEN |

---

## TODO 주석 위치

| 항목 | 파일:라인 | 통합 시점 |
|------|----------|----------|
| W-3 | `workers/inspect.py:127` | known_hosts 명시 경로 |
| W-3 | `workers/sw_install.py:109` | known_hosts 명시 경로 |
| C-2 | `api/websocket.py:35` | W-5 API 인증 구현 시 |

---

## 미처리 블로커 (우선순위 순)

1. `config/logging.py` 신규 — structlog 민감필드 마스킹 (Blocker 9) ← **다음 작업**

---

## 10차 세션 완료 항목

### PR #30 — Blocker 7b + Blocker 8

- **workers/sw_install.py** 신규 (q_sw_install Celery 태스크)
  - 표준 sys_config 항상 적용 (Ubuntu 직접 / 비Ubuntu → SW Planner Agent):
    GRUB 파라미터(CPU 종류별), CPU 거버너 performance, GPU PM(NVIDIA), 자동 업데이트 방지
  - 13종 패키지 설치 함수 (sw-install.md 절차 준수):
    nvidia_driver/cuda/cudnn/torch/docker/docker_container_toolkit/
    miniconda/python/gcc/rustup/tt_kmd/tt_smi/tt_burnin
  - nvidia-driver 설치 후 reboot → 300s SSH 재접속 폴링 → driver 검증
  - account 생성 / storage_mount / 비정형 sys_config (→ Agent 위임)
  - 설치 실패 → SW Planner Agent 복구 시도, 복구 불가 → job FAILED + cleanup
  - 의존성 순서 정렬 `_sort_items` + pre/post reboot 분리 `_split_items_by_reboot`
- **workers/inspect.py**: C-5 stub 제거 → sw_requirements 유무로 분기
  (`run_sw_install(q_sw_install)` / `run_post_install(q_inspect)`)
- **workers/app.py**: `workers.sw_install` include 추가
- **config/celeryconfig.py**: `workers.sw_install.*` 라우팅 활성화
- **tests/test_workers/test_sw_install.py**: 25개 테스트

---

## 미처리 WARNING

W-3 `known_hosts=None`→명시 경로 / W-4 Redis 인증 / W-5 API 인증 / W-7 `raw_output` 누락 / W-8 SSH mock / W-10 `create_type=False`

---

## 세션 기록

- [5차 세션](session-archive/session-5.md) — 문서 교차검증 2라운드, PR #10~14 머지 완료, PR #16 대기
