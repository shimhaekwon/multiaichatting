# Multi-LLM 협업 플랫폼 — 대화 정리 (2차)

---

## 핵심 시스템 정의

> 브라우저 로그인 세션 기반으로 접근 가능한 LLM들을 탐지·선택하고, 동기식 3라운드 토론을 거쳐 사용자가 최종 판단하는 **Human-in-the-Loop 멀티 LLM 협업 플랫폼**

---

## 핵심 규칙

| 항목 | 내용 |
|---|---|
| LLM 접속 | 브라우저 로그인 세션 (API Key 불필요) |
| LLM 선택 | 로그인된 LLM 목록에서 일부 또는 전체 선택 |
| Round 1 | 독립(Blind) 응답 |
| Round 2 | 상호 검토·비판 |
| Round 3 | 최종 제안 |
| 동기화 | 모든 LLM 응답 완료 후 다음 라운드 진행 |
| 최종 판단 | 사용자 직접 (AI 자동 합의 없음) |

---

## 사용자 권한

- 각 라운드마다 개입 가능
- 특정 LLM에게 1:1 근거 재질의 (Deep Dive)
- 최종 Commit은 사용자가 수행

---

## 설계 원칙

- Context 폭발 방지 (압축 전달)
- Timeout 15초 → 미응답 LLM DROP 후 진행
- 응답은 간결하게, 상세는 Deep Dive로

---

## 산출물 로드맵

```
요구사항정의서 → 시스템 설계서 → 상세설계서 → 구현
```

---

## ChatGPT의 핵심 지적

"No more question" 수준 문서가 되려면 아래 6개 층이 모두 필요:

1. **Product Spec** — 대상 사용자, MVP 범위, 제외 범위
2. **Functional Spec** — 라운드 조건, timeout, retry, failover 등
3. **State Machine** — IDLE → ROUND_RUNNING → BARRIER_WAIT → HITL_WAIT → FINAL → COMPLETED
4. **API Contract** — field 단위 타입·enum·validation·error code
5. **Infrastructure Spec** — 배포 환경, 스케일링 방식
6. **NFR** — latency, 보안, logging, 세션 제한

---

## 현재 상태

- 요구사항 방향 확정 ✅
- 기술 스택 미확정 (Extension? Electron? Vue? React?)
- 다음 단계: 미확정 항목 결정 후 요구사항정의서 완성