# Handoff — Inspection System v2

> 최종 업데이트: 2026-04-16 (17차 세션)
> 이 파일의 범위: **다음 작업 + 블로커 + WARNING** 만. 아키텍처·구현 현황 → 프로젝트 `CLAUDE.md`

---

## 추천 시작점

**다음 작업**: 리포트 개편 PR C — 차트 삽입 (matplotlib → PNG → LaTeX)

**병행 대기**:
- PR #39 (feat/report-structure) — 리뷰 반영 완료, merge 대기
- WebGUI 프론트엔드 (보류 — 기술 방향 미확정)

**전제조건 확인**:
```bash
git checkout main && git pull
pytest tests/ -x -q   # 278 passed, 8 skipped
ruff check . && ruff format --check .
```

---

## 리포트 개편 작업 계획 (진행 중)

### PR 진행 현황

| PR | 브랜치 | 내용 | 상태 |
|----|--------|------|------|
| #38 | fix/report-p0-bugs | P0 버그 3건 수정 | **MERGED** |
| #39 | feat/report-structure | PR B — 구조 개편 | OPEN (리뷰 반영 완료) |
| — | — | PR C — 차트 삽입 | 미시작 |
| — | — | PR D — 신규 섹션 | 미시작 |
| — | — | PR E — TikZ/tcolorbox (옵션) | 미시작 |

### PR #38 완료 내용 (MERGED)

1. `templates/report.tex.j2:131` `"\textemdash"` → `"—"` (U+2014) 이중이스케이프 수정
2. `_detail_latex` 필터: `|` 기준 분리 → `\newline` 재결합 (상세 컬럼 잘림 수정)
3. `report.py` `fail_items`/`warn_items` 키 불일치 수정 (판정 섹션 빈칸)
4. fallback `overall` 수정: `"fail" if fail_reasons else "warn" if warn_reasons else "fail"` (PR #39에 포함)

### PR #39 완료 내용 (OPEN — merge 대기)

**`workers/report.py`**:
- `KNOWN_FIELDS`: 14개 스크립트별 표시 필드 목록
- `parse_detail()`: `"key=val|key2=val2"` → dict
- `_build_phase_map()`: profile JSON에서 script→phase 역방향 매핑 자동 생성
- `_enrich_results()`: check_results에 `phase`, `display_fields` 보강
- `_fields_latex()` Jinja2 필터 추가
- context: `preflight_results`, `post_install_results`, `collect_results`, `unknown_results`, `pass_count`, `warn_count`, `fail_count`
- XLSX: Phase 컬럼 추가 (col 2)
- fallback overall 버그 수정 (PR #38 리뷰 반영)

**`templates/report.tex.j2`** (전면 재작성):
- Executive Summary: PASS/WARN/FAIL 카운트 카드
- Phase 섹션 헤더: Preflight(#1A5C8A) / Post-install(#2D6A4F) / 로그 수집(#5C4B1A) / 기타(#555555)
- `render_phase_table` Jinja2 macro
- 상세 컬럼: `display_fields | fields_latex` 필터
- REJECTED 배지: rejectedbg(#FFE0B2)/rejectedtext(#E65100) 주황색
- `\statusbadge` 데드코드 삭제

### PR C — 차트 삽입 (다음 작업)

```
workers/report.py에 _generate_charts() 추가
  - chart_gpu_stress.png: peak_temp/power/util 3-bar (matplotlib)
  - chart_nccl.png: bw_2gpu/bw_4gpu vs 기준선
  - chart_status_pie.png: PASS/WARN/FAIL 도넛
templates/report.tex.j2에 \includegraphics 삽입
```

### PR D — 신규 섹션

```
- H/W 수동 검수 (Section 2): jobs.hw_manual_checks JSON 8항목 체크리스트 + 서명란
- sys-config 적용 결과 (Section 6): GRUB/governor/PM/auto_update 체크박스 테이블
- Agent 사용 내역 (Section 5): Inspect/Verify/SW Planner 호출 여부 + 사유
- 부록 — 로그 경로 안내 (Section 7)
```

### 관련 파일

| 파일 | 역할 |
|------|------|
| `workers/report.py` | Celery task, context 조립, xelatex 컴파일 |
| `templates/report.tex.j2` | LaTeX 템플릿 |
| `checks/profiles/gpu_server.json` | `validation.rules` (기준값 출처) |
| `.claude/rules/report.md` | 리포트 규칙 (목표 스펙) |

### 실서버 테스트 job_id 참고

- 최종 성공: `87303440-60f1-4773-acac-068e5cbb7482` (RTX 4090 x4, PR #37 검증용)
- 리포트 경로: `/srv/inspection/results/{job_id}/`
- 다운로드: `curl -sL http://localhost:8000/api/reports/{job_id}/pdf -o report.pdf`

---

## 열린 PR 목록

| PR | 브랜치 | 내용 |
|----|--------|------|
| #39 | feat/report-structure | PR B — 구조 개편, merge 대기 |

최근 머지: #33, #34, #35, #36, #37, #38

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
| report 구조 개편 (Phase분리/Executive Summary/KNOWN_FIELDS) | #39 | 🔄 merge 대기 |
| **리포트 차트 삽입** | — | ☐ |
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
