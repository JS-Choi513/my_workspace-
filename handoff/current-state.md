# Handoff — 멀티프로젝트 인덱스

> 워크스페이스는 멀티프로젝트 컨테이너. 각 프로젝트의 실제 진행 상태·다음 작업·블로커는 프로젝트 내부 handoff에 기록.
> 이 파일은 **각 프로젝트 handoff의 위치만** 가리킨다.

---

## 프로젝트별 handoff 위치

| 프로젝트 | 현재 상태 | handoff 경로 |
|---------|----------|-------------|
| inspection-system | v2 리팩토링 진행 중 | `projects/inspection-system/handoff/current-state.md` |
| p2p-perf-monitor | Phase 4 진행 중 (Docker Compose + systemd 패키징) | `projects/p2p-perf-monitor/handoff/current-state.md` |
| gadgetron | 외부 repo clone (Claude 하네스 미정비) | `projects/gadgetron/README.md` |

---

## 새 세션 시작 시

1. 어떤 프로젝트로 작업할지 사용자에게 확인 또는 사용자 지시에 따라 결정
2. 해당 프로젝트의 handoff 파일 + `CLAUDE.md`를 읽어 컨텍스트 적재
3. 워크스페이스 공통 규칙은 `rules/`·`CLAUDE.md` 참조 (프로젝트 규칙이 우선)

---

## 워크스페이스 자체 진행 항목

이 워크스페이스 하네스(인덱스·공통 규칙·템플릿) 자체에 대한 변경 사항이 있을 때 기록.

- 2026-05-11: 멀티프로젝트 구조로 재구성. inspection-system 전용 context/handoff 회수, rules 중립화.
