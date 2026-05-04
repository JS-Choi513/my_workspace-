# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-23
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 작업**: PR D — 리포트 신규 섹션 (H/W 수동검수, sys-config 결과, Agent 사용내역, 로그 위치 안내)

**현재 상태**:
- `main` tip: `3193855` (PR #45 flower 의존성 추가)
- PR #43·#44·#45 모두 머지 완료 (2026-04-23)
- PR #44 실서버 E2E 검증 완료 (Job `0d01e2e5`, gadget-burn 교체 확인)
- 열린 PR 없음, uncommitted 없음

**다음 액션**:
1. PR D 범위 확정 (리포트 섹션 4종: H/W 수동검수 / sys-config / Agent 사용내역 / 로그 안내)
2. 브랜치 생성 `feat/report-additional-sections` 등
3. `pytest tests/ -x -q` + `ruff check . && ruff format --check .`
4. `gh pr create`

**병행 대기**:
- WebGUI 프론트엔드 (보류 — 기술 방향 미확정)

**환경 주의**:
- `COMPOSE_FILE=/opt/inspection-system/inspection-system/docker-compose.yml` 환경변수가 shell에 남아있으면 **구버전 deprecated 경로로 redirect**되어 compose 명령이 오작동. `unset COMPOSE_FILE` 필요. `/opt/...` 경로 deprecated, 전부 무시.

**전제조건 확인**:
```bash
unset COMPOSE_FILE
git checkout main && git pull
pytest tests/ -x -q   # 278 passed, 8 skipped 기대
ruff check . && ruff format --check .
```

---

## 19차 세션 (2026-04-23) 완료 항목

### PR #43 (MERGED) — PR #42 리뷰 대응 (CodeQL 경고 정리)
- 2 commits: `cd04b4e` + `2687e6c`
- github-code-quality bot 5건 "Empty except" 경고 해소
- `stress_gpu.py`의 `except (ValueError, TypeError): pass` 5개 블록에 `# [N/A] 등 비정상 문자열 시 ... 무시 (베스트에포트)` inline 주석 추가

### PR #44 (MERGED) — `gpu_burn` → `gadget-burn` 도구 교체
- 1 commit: `ea52f63`
- `wilicc/gpu-burn` → `DEEPGadget/gadget-burn` (MIT, 사내 도구, NVIDIA Pascal~Blackwell)
- 바이너리 경로·CLI 통일 (`-t <dur>` 공통, gaming/datacenter 분기 제거)
- `tool` 필드 값 `gpu_burn` → `gadget-burn` (스키마 변경 없음)
- 6 파일, +46 -51

### PR #45 (MERGED) — flower 의존성 추가
- 1 commit: `3193855`
- `pyproject.toml`에 `flower>=2.0` — celery 서브커맨드 `flower` 없이 컨테이너 exit 2 하던 문제 해소
- `:5555` HTTP 200 검증 완료

### 실서버 E2E 검증 — PR #44 검증 (Job `0d01e2e5-1247-42a1-9979-48e8a6daeb1c`)
- 10.100.1.23 (AMD EPYC 7313 x2 + RTX 4090 x4), 2026-04-23 16:50 KST
- 소요: 약 10분, 14 checks (pass 9 / warn 3 / fail 2), 판정: **FAILED**
- **gadget-burn 교체 핵심 검증**:
  - `tool=gadget-burn` 필드가 inspect_raw JSON + 리포트 PDF 상세 컬럼 모두에 정확히 표시
  - `duration_s=300` CLI 옵션 정상 인식
  - `gpu_class=gaming` (RTX 4090 detail 표시용 유지 확인)
  - 피크 70°C, 풀로드 95%, HW throttle 0, ECC 0 — 스트레스 데이터 수집 정상
- **실서버 자체 이슈 재현** (이전 Job b69ce10d와 일관):
  - `nccl_bandwidth.bw_2gpu_gbs=10.24` (이전 10.38, 오차 1.4%) → FAIL
  - `nccl_bandwidth.bw_4gpu_gbs=4.93` (이전 5.04, 오차 2.2%) → FAIL
  - `stress_gpu.cooling_consistency_score=34` (GPU 간 초반 peak spread 25°C) → FAIL
- 리포트 PDF: `~/report_gadget_burn.pdf` (284 KB, 4페이지) — 시계열 차트·heatmap·NCCL bar 정상

### 배포 워크플로우 관련 학습
- **`COMPOSE_FILE` 환경변수**가 구버전 `/opt/inspection-system/inspection-system/docker-compose.yml` 가리키고 있어 `docker compose` 명령이 구버전 경로로 redirect됐음
- `/opt/...` 경로는 **deprecated**. `unset COMPOSE_FILE` 후 `/home/dg/workspace/projects/inspection-system/`에서 배포
- 증상: `worker_sw_install` 서비스 파싱 누락, alembic versions 3개 누락 — 모두 같은 원인의 파생
- 영구 해결: shell rc 파일에서 `COMPOSE_FILE` export 제거 (사용자 조치 사안)

---

## 18차 세션 (2026-04-22) 완료 항목

### PR #42 (MERGED 2026-04-22) — 부하 테스트 시계열 + 차트 + 표시 개선
4 commits: `4f16756` + `591ba3e` + `40ead6f` + `6747b05`

**시계열 로깅 + matplotlib 차트**:
- `stress_gpu.py` (+279 LOC): nvidia-smi 샘플 → `{job_id}/inspect_raw/stress_gpu_timeseries.jsonl`
- `stress_cpu.py` (+69 LOC → 멀티소켓 분리): 5초 loop 샘플 → `stress_cpu_timeseries.jsonl`
- `workers/report_charts.py` 신규 (~600 LOC): matplotlib 3-panel 시계열 + NCCL bar + status donut
- `workers/report.py` (+100 LOC): `_generate_charts()` 통합
- `templates/report.tex.j2`: `\includegraphics` 삽입
- `pyproject.toml`: matplotlib 의존성 추가
- 냉각 일관성 지표 포함

**CPU 멀티 소켓 + 폰트 정책**:
- AMD EPYC 2소켓 등 소켓별 시계열 분리
- `Dockerfile`: Roboto/Consolas 폰트 설치
- `templates/report.tex.j2`: fontspec 설정

**리포트 표시 개선**:
- `display_name` 14종 매핑 (스크립트명 → 한글)
- `sw_gpu_hw.py` (+115 LOC): PCIe 인식 수정 (sudo 경유 `lspci -vv`, 모든 GPU 조사, LnkCap + LnkSta 병기)
- 상세 컬럼 폰트 `\scriptsize\ttfamily`로 축소

### 리뷰 반영 (`cd04b4e`)
- PR #42 리뷰 4건 — CodeQL 경고 정리

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

## 향후 PR 계획

| PR | 내용 | 우선순위 |
|----|------|---------|
| D | 신규 섹션 (H/W 수동검수, sys-config, Agent 사용내역, 로그 안내) | 다음 |
| E (옵션) | TikZ Phase 파이프라인 흐름도, tcolorbox | 여력 있으면 |

---

## 열린 PR 목록

없음 (모든 PR 머지 완료).

최근 머지: #37, #38, #39, #40, #41, #42, #43, #44, #45

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
| report 리뷰 반영 + KST + 진단메시지 + AMD CPU 온도 | #40 | ✅ |
| CI 자동 리뷰 워크플로우 + OIDC 권한 수정 | #41 | ✅ |
| 부하 테스트 시계열 + matplotlib 차트 + 멀티소켓/폰트/표시 개선 | #42 | ✅ |
| PR #42 리뷰 대응 (CodeQL empty except 5건) | #43 | ✅ |
| `gpu_burn` → `gadget-burn` 도구 교체 + 실서버 E2E 검증 | #44 | ✅ |
| flower 의존성 추가 (celery 서브커맨드 복구) | #45 | ✅ |
| **리포트 신규 섹션** (H/W 수동검수, sys-config, Agent 내역, 로그 안내) | — | ☐ |
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
