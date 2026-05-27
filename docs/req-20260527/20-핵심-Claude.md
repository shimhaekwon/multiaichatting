## 1. 시스템 아키텍처

### 1.1 베이스
- **Base**: ChatHub (E:\workspace\chathub) fork & enhancement
- **Type**: Chrome Extension (Manifest V3)
- **Storage**: IndexedDB (local-only, no server)
- **Auth**: Browser session cookie 기반, API key 미사용

### 1.2 LLM Adapter Layer
- Claude / ChatGPT / Gemini 각각 web session adapter 구현
- 로그인 상태 detection: content script로 각 사이트 cookie/session 확인
- 미로그인 시: 해당 LLM 로그인 페이지 안내 → 유효 LLM 0개 시 default LLM 계정 생성 가이드

## 2. 세션 & 선택 관리

| 항목 | 사양 |
|---|---|
| 최소 선택 LLM | 2개 (validation 강제) |
| 선택 상태 persistence | chrome.storage.local |
| 라운드 도중 변경 | 차단, 라운드 경계에서만 mutation 허용 |

## 3. 라운드 엔진

### 3.1 라운드 정의
```
R1 (Blind):       사용자 프롬프트만 전달, 타 LLM 응답 미공유
R2 (Review):      R1 전체 응답 + 사용자 프롬프트 전달, 상호 검토
R3 (Final):       R2까지 누적 컨텍스트 + 최종 답변 지시
R4~R9 (Extended): 사용자 연장 시 동일 패턴 반복
```

### 3.2 동기화
- 모든 선택된 LLM의 응답 완료 이벤트를 Promise.all 패턴으로 대기
- 한 LLM이라도 미완료 시 다음 라운드 진입 불가
- 타임아웃 도달 시 retry (3~9회 사용자 조정)

### 3.3 라운드 제어
- 기본 3회, 최대 9회, ±1 단위 증감 (마지막 회차에서도 +1 가능)
- 조기 종료: 사용자 명령 시 즉시 종료
- 자동 종료: 지정 회차 완료 시점, 사용자 연장 prompt 표시

## 4. Request Payload Spec
각 LLM에 전달하는 메시지에 포함:
- 사용자 원본 prompt
- 현재 라운드 / 최대 라운드 (예: `R2/3`)
- 타임아웃 (분 단위)
- 직전 라운드 타 LLM 응답 (R2 이후)
- 출력 형식 강제 지시문 (structured, ≤1500 tokens)

## 5. 사용자 개입 정책

| 시점 | 허용 동작 |
|---|---|
| Request 후 ~ 전체 Response 완료 전 | 개입 불가 |
| 라운드 사이 | LLM 추가/제외, 1:1 재질의, 회차 증감, 조기 종료 |
| Response 진행 중 | 다음 회차 prompt 사전 입력 가능 (send buffer만, 전송 차단) |
| MVP 이후 | 선응답 LLM과 개별 "몰래 채팅" |

재질의 결과 → 다음 라운드 컨텍스트에 자동 주입

## 6. 저장 스키마

### 6.1 IndexedDB Object Stores
- `sessions` (id, createdAt, llms[], maxRounds, ...)
- `rounds` (sessionId, roundNo, status, startedAt, completedAt)
- `responses` (sessionId, roundNo, llm, content, tokens, latency)
- `userInputs` (sessionId, roundNo, prompt, sideQueries[])

### 6.2 Export
- 전체 통합 MD 1개 + LLM별 개별 MD N개
- 파일명 컨벤션: `{YYYYMMDD-HHMM}_{sessionId}_{llm}.md`

### 6.3 History 검색
- Index: createdAt(년월일시) / llm / roundNo

## 7. 출력 강제 규칙
- System prompt에 structured output 지시 주입
- 1500 token 상한 (truncation 또는 prompt-level 제한 — 설계 단계에서 결정 필요)
- 장황한 prose 회피 지시

## 8. 비기능 요구
- Chrome 최신 안정 버전
- Windows / macOS / Linux
- 단일 사용자, 암호화 미적용, 모바일 미지원

## 9. 개발 원칙
- **step by step**: 라운드/모듈 단위 점진 구현
- **no guess**: 합의되지 않은 사양 추정 구현 금지 → 명세 확정 후 코드
- **4무 원칙**: 빠짐없이 / 틀림없이 / 다름없이 / 더함없이
- **재사용 자산화**: adapter, round engine, storage layer 템플릿화

## 10. 미결 항목 (확정 필요)
- 1500 token 제한 enforcement 방식 (post-truncate vs prompt-limit)
- LLM adapter의 DOM 변경 대응 전략 (selector 버저닝)
- 재시도 시 idempotency 보장 방식
