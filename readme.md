# Multi-LLM Cowork System (다자 AI 협업 시스템)

여러 LLM(Claude, ChatGPT, Gemini)과 사용자가 동시에 협업하는 **브라우저 세션 기반 멀티 LLM 협업 플랫폼**.

## 개요

사용자가 크롬 브라우저에 로그인된 계정을 활용하여 접속 가능한 LLM들을 탐지·선택하고, 
동기식 3라운드 협업(R1: 독립응답 → R2: 상호검토 → R3: 최종답변)을 거쳐 
사용자가 최종 판단을 내리는 HITL(Human-in-the-Loop) 시스템입니다.

**핵심 특징:**
- API 키 불필요 (크롬 로그인 세션 활용)
- Claude / ChatGPT / Gemini 동시 협업
- 최대 9라운드까지 사용자 설정 가능
- LLM별/전체 MD 파일 저장
- IndexedDB 기반 로컬 자동 저장

## 프로젝트 구조

```mermaid
graph TD
    START["start/"] --> TODO["todo.md - 사용자 요구사항 원본"]
    START --> BS["brain-storming/ - LLM별 초기 브레인스토밍"]
    
    DOCS["docs/"] --> REQ["req/ - 각 LLM별 요구사항 초안 v0.1"]
    DOCS --> REQ_FINAL["req-final/ - 최종 확정 요구사항정의서 v1.0"]
    DOCS --> DESIGN["design-ui_ux_api/ - 설계 방향 논의 및 3사 통합안"]
    
    REQ_FINAL --> CLAUDE["claude.md"]
    REQ_FINAL --> GEMINI["gemini.md"]
    REQ_FINAL --> CHATGPT["chatgpt.md"]
    REQ_FINAL --> DEEPSEEK["deepSeek.md"]
    REQ_FINAL --> MISTRAL["mistral.md"]
```

## 요구사항 (확정)

| 구분 | 내용 |
|------|------|
| 대상 LLM | Claude, ChatGPT, Gemini (3종) |
| LLM 접속 | 크롬 로그인 세션 기반 (API 키 불필요) |
| LLM 선택 | 최소 2개 이상, 세션 간 선택 상태 유지 |
| 라운드 | 기본 3회, 사용자 설정 시 최대 9회 |
| R1 | Blind 독립 응답 |
| R2 | 타 LLM 응답 전체 텍스트 검토 |
| R3 | 최종 답변 |
| 동기화 | 모든 LLM 응답 완료 후 다음 라운드 진행 |
| 타임아웃 | 5분 (LLM 무응답 시 제외 후 진행) |
| 사용자 권한 | 단계별 개입 / 1:1 재질의 / 최종 판단 |
| 저장 | IndexedDB 자동 저장 + MD 파일 저장 (전체/LLM별) |
| 구현 환경 | Chrome Extension + Web UI |

## 협업 흐름

```mermaid
sequenceDiagram
    participant U as 사용자
    participant S as 시스템
    participant C as Claude
    participant G as ChatGPT
    participant GM as Gemini

    U->>S: 질문 입력
    S->>C: R1: Blind 독립 응답
    S->>G: R1: Blind 독립 응답
    S->>GM: R1: Blind 독립 응답
    Note over S: 동기화 배리어 (모든 응답 완료 대기)
    C-->>S: R1 응답
    G-->>S: R1 응답
    GM-->>S: R1 응답
    
    U->>S: 검토 및 개입 (선택사항)
    
    S->>C: R2: 타 LLM 응답 검토
    S->>G: R2: 타 LLM 응답 검토
    S->>GM: R2: 타 LLM 응답 검토
    Note over S: 동기화 배리어
    C-->>S: R2 검토의견
    G-->>S: R2 검토의견
    GM-->>S: R2 검토의견
    
    U->>S: 1:1 재질의 (선택사항)
    
    S->>C: R3: 최종 답변
    S->>G: R3: 최종 답변
    S->>GM: R3: 최종 답변
    C-->>S: R3 최종답변
    G-->>S: R3 최종답변
    GM-->>S: R3 최종답변
    
    U->>S: 최종 판단
```

## 진행 상태

- [x] 브레인스토밍 및 아이디어 탐색
- [x] 요구사항정의서 v1.0 확정
- [ ] 시스템 설계
- [ ] PoC 구현
- [ ] MVP 구현

## 개발 원칙

- **Step by step**: 단계별 검증 후 진행
- **No guess**: 합의되지 않은 내용은 추정 구현 금지
- **빠짐없이 / 틀림없이 / 다름없이 / 더함없이**
- 재사용 가능한 설계 템플릿 자산화

## 참고

- UI/UX 참고: [ChatHub](https://chathub.gg) (크롬 확장 프로그램)
