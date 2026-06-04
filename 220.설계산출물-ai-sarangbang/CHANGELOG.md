# 변경 이력 — 220.설계산출물-ai-sarangbang (CHANGELOG)

> 설계산출물 폴더의 **시간순 변경 이력**(사람이 읽는 의미 단위). git 커밋과 별개로 **설계 의사결정·문서 변경**을 추적한다.
> - 각 문서의 **현재 버전** = [`README.md`](./README.md) 문서목록 표.
> - 각 문서의 **내부 상세 이력** = 해당 문서 말미 "변경 이력" 섹션.
> - 표기: `문서 — 버전 (구분)` + 요약. **최신 날짜가 위로**.

---

## 2026-06-04

### 228-진입로비-런타임모델선택.md — **v1.0 (신규)**
- **신규 설계 변경안**: 진입 시 **로비**에서 로컬 Ollama 모델 목록(`/api/tags`)을 보여주고 AI별 **모델·이름·사고모드 선택 + 인원 가감** → 입장. **localStorage 영속**.
- 사용자 희망 1~5(미실행 처리·thinking·영속화·인원가변·이름동기화) 전부 반영.
- **최종 검토 반영(FIX-FIRST)**: uw-critic + 코드 정합 교차 → **C1**(`/ollama` proxy dev 전용 → preview.proxy)·**C2**(세션 teardown `Coordinator.dispose()` 부재·StrictMode 누수)·**H1**(buildSession 조립 verbatim)·**H2**(Mock 폴백 id 불일치)·**H3**(영속 스키마 버전·검증)·M1~4·L1~3.
- 핵심: AgentDriver+participants 아키텍처는 건전, 수정은 teardown·proxy 범위·조립 재현. 상태: **구현 대기**.

### 227-설계변경-floor-random직렬화-및-순서표시.md — **v1.0 (신규)**
- **신규 설계 변경안**: race(속도 경쟁 floor) → **랜덤 순서 직렬** 발언 + **Roster 순번 배지**.
- **트리거**: 라이브 디버깅 — `EXAONE`·`Qwen3` 연속 "응답 없음".
  - 원인(측정): 전 모델 **100% CPU** · RAM **free 7.6GB < 워킹셋 14GB** · 동시 race가 자원 경쟁 → 작은 `phi4` 독식, 큰 모델 굶어 60초 타임아웃.
- **확정 결정**: D-A=**B만**(순번 배지) · D-B=**코드만 먼저** · D-C=MD 미포함 · D-D/E=후속.
- **최종 검토 반영(FIX-FIRST)**: 독립 `uw-critic` 적대 검토 + 작성자 코드 정합 교차검증 → **C1**(onOrder 배선 부재)·**C2**(배지 생명주기)·**H1**(speakOne)·**H2**(빈응답 테스트 보존)·**H3**(core-purity `Math.random` 가드)·**M1~4**·**L1~3** 전량 반영.
- **영향 예정 정본**: [221]§R2 · [222] · `core-purity.test` (구현 후 갱신).
- 상태: **구현 대기**.

### (체계) 버전 이력 관리 도입
- 본 `CHANGELOG.md` 신설 + `README` 문서목록에 **버전·수정일 컬럼** 추가 + 각 문서 말미 **"변경 이력" 섹션 표준화**(227부터 적용).

---

## 2026-06-02 (baseline)

### 221 ~ 226 — **v1.0 (정본 확립)**
- ai-sarangbang 요구사항·설계 정본 문서군 작성·2차 검토·origin push(`467aa70`).
- 221 제품개요·핵심규칙(R1~4·D1~5) / 222 coordinator 상태머신(single-flight·whisper·driver seam) / 223 데이터모델·로그포맷 / 224 아키텍처·스택·단계 / 225 UI-UX(인터랙티브 목업·O1) / 226 구현계획-P0.
- 상세 처리내역: `E:\프로젝트\multiaichatting\900.AICOWORK\20260602\`.
