# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-17 (17차 세션, 3회차)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 작업**: 부하 테스트 시계열 로깅 + 리포트 차트 삽입 (PR C)

**병행 대기**:
- WebGUI 프론트엔드 (보류 — 기술 방향 미확정)

**전제조건 확인**:
```bash
git checkout main && git pull
pytest tests/ -x -q   # 278 passed, 8 skipped
ruff check . && ruff format --check .
```

---

## 17차 세션 (2026-04-16) 완료 항목

### PR #38 (MERGED) — P0 버그 3건
1. `\textemdash` 이중이스케이프 → U+2014 em dash 직접 사용
2. 상세 컬럼 줄바꿈 (`detail_latex` 필터)
3. `fail_items`/`warn_items` 키 불일치 수정

### PR #39 (MERGED) — 구조 개편
- Executive Summary: PASS/WARN/FAIL 카운트 카드
- Phase별 섹션 분리 (Preflight/Post-install/로그수집/기타)
- `KNOWN_FIELDS` 기반 구조화된 상세 표시 (14개 스크립트)
- `parse_detail()`, `_build_phase_map()`, `_enrich_results()` 추가
- `render_phase_table` Jinja2 macro
- fallback `overall` 로직 수정 (warn-only/empty 케이스 오판정 방지)

### PR #40 (MERGED) — 리뷰 반영 + 실서버 검증 피드백
**리포트**:
- REJECTED 주황색 (failbg와 구별), preflight 색상 중복 수정, `\statusbadge` 데드코드 삭제
- 시간 표시 UTC → **KST** 변환 (`astimezone(KST)`)
- `parse_detail()`: `WARN:/FAIL:/INFO:` 진단 메시지 별도 수집
- `_fields_latex()`: WARN=주황/FAIL=빨강/INFO=파랑 색상 강조
- `KNOWN_FIELDS`: stress_gpu/stress_cpu/nccl_bandwidth에 `tool`, `duration_s` 추가 (실행 여부 명시)
- `KNOWN_FIELDS.nccl_bandwidth`: `min_bw_*_gbs`, `gpu_count` 추가

**AMD CPU 온도 센서 지원** (실서버 AMD EPYC 7313 × 2 검증):
- `sw_cpu.py`: hwmon sysfs 우선 탐색 (`coretemp` Package / `k10temp` Tctl-Tdie)
- `stress_cpu.py`: 동일하게 hwmon + sensors 정규식에 `Tctl|Tdie` 추가
- `peak_temp=0` (측정 실패) → WARN 승격 + `peak_temp_c=unknown` 표시

### PR #41 (MERGED) — CI 자동 리뷰 워크플로우
- 자동 리뷰 워크플로우 추가 + 기존 workflow 정리
- `id-token: write` 권한 추가 (OIDC 토큰 인증 실패 수정)

### 실서버 검증 기록
- Job `b69ce10d-3bdf-4568-ad9c-958143371c36` (10.100.1.23, 2026-04-16 13:41 KST)
- 소요: 10분, 14 checks, 판정: **FAIL**
- 검증된 항목:
  - gpu_burn 정상 실행 (peak_temp_c=73, power_ratio=100%, avg_util=96%)
  - nccl-tests 정상 실행 (bw_2gpu=10.38, bw_4gpu=5.04)
  - FAIL 사유: `nccl_bandwidth.bw_2gpu_gbs=10.38 (fail_below=30)` — 2 GPU all_reduce 기준 미달
- 리포트 PDF: `~/report_full_test.pdf`

---

## 다음 작업 — PR C (차트 삽입)

사용자 피드백: **"부하테스트 시계열 그래프 필요"** — bar chart 아닌 time series plot 필요.

### 작업 분할

**Step 1: 시계열 데이터 수집**
- `stress_gpu.py`: nvidia-smi 샘플을 `{job_id}/inspect_raw/stress_gpu_timeseries.jsonl`에 append
  - 필드: `(timestamp, temp_c, power_w, util_pct, freq_mhz, per_gpu_dict)`
- `stress_cpu.py`: 기존 5초 간격 loop에서 샘플을 `stress_cpu_timeseries.jsonl`에 append
  - 필드: `(timestamp, peak_temp_c, freq_mhz, util_pct)`
- `nccl_bandwidth.py`: all_reduce_perf 단일 측정이라 시계열 아님 → bar chart로 처리

**Step 2: matplotlib 차트 생성**
- `workers/report.py`에 `_generate_charts(job_id, context)` 추가
- `chart_gpu_stress.png`: 시계열 (온도/전력/사용률 3-panel)
- `chart_cpu_stress.png`: 시계열 (온도/주파수/사용률 3-panel)
- `chart_nccl.png`: bar chart (bw_2gpu/bw_4gpu vs 기준선)
- `chart_status_donut.png`: PASS/WARN/FAIL 도넛

**Step 3: LaTeX 템플릿**
- 각 stress 섹션 뒤에 `\includegraphics` 삽입
- Executive Summary에 status donut 삽입

### 관련 파일

| 파일 | 역할 |
|------|------|
| `checks/base/post_install/stress_gpu.py` | nvidia-smi 샘플링 (시계열 로깅 추가 필요) |
| `checks/base/post_install/stress_cpu.py` | 기존 5초 loop (샘플 JSONL 저장 추가) |
| `workers/report.py` | matplotlib chart 생성 함수 추가 |
| `templates/report.tex.j2` | `\includegraphics` 삽입 |
| `pyproject.toml` | `matplotlib` 의존성 추가 필요 |

---

## 향후 PR 계획

| PR | 내용 | 우선순위 |
|----|------|---------|
| C | 부하 테스트 시계열 로깅 + 차트 삽입 | 다음 |
| D | 신규 섹션 (H/W 수동검수, sys-config, Agent 사용내역, 로그 안내) | C 이후 |
| E (옵션) | TikZ Phase 파이프라인 흐름도, tcolorbox | 여력 있으면 |

---

## 열린 PR 목록

| PR | 브랜치 | 내용 |
|----|--------|------|
| #40 | feat/report-refinements-and-amd-temp | 리포트 개선 + AMD 온도, merge 대기 |
| #41 | ci/auto-review-workflow | 자동 리뷰 워크플로우 추가 + claude-fix 정리 |

최근 머지: #33, #34, #35, #36, #37, #38, #39

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
| harness 문서 갱신 | #36 | ✅ |
| GPU 분류 + preflight 버그 4건 + SSH env transport | #37 | ✅ |
| report P0 버그 3건 (textemdash/컬럼잘림/verdict빈칸) | #38 | ✅ |
| report 구조 개편 (Phase분리/Executive Summary/KNOWN_FIELDS) | #39 | ✅ |
| report 리뷰 반영 + KST + 진단메시지 + AMD CPU 온도 | #40 | 🔄 merge 대기 |
| **부하 테스트 시계열 + 차트** | — | ☐ 다음 작업 |
| **리포트 신규 섹션** (H/W 수동검수 등) | — | ☐ |
| **WebGUI 프론트엔드** | — | ☐ 보류 |

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
