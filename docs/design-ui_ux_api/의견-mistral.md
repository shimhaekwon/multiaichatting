아래는 **"No-Question Build"를 위한 최적화된 설계 문서 구조**와 **실제 구현 시 공수 절감 방법**을 **현실적인 PM 관점**에서 정리한 최종 가이드입니다.
**70% 설계 + 30% 구현 보완** 전략을 **구현 가능한 수준**까지 구체화합니다.

---

## **1. 최적화된 문서 구조 (재사용성 + 공수)**
### **필수 문서 (구현 전 90% 이상 필수)**
| **파일명**               | **내용**                                                                 | **공수 절약 방법**                                                                 |
|---------------------------|--------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| **`requirements.md`**     | - **핵심 요구사항** (LLM 3개, R1/R2/R3 라운드)                       | Markdown + GitHub Wiki (편집 속도 ↑)                                               |
| **`architecture.md`**    | - **시스템 아키텍처** (Extension + Web UI + DB)                         | TypeScript `interface`로 DB/API 스펙 통합 (코드 + 문서)                             |
| **`protocol.md`**         | - **LLM 협업 프로토콜** (R1/R2/R3 로직)                               | 의사코드 + 상태전환 다이어그램 (간결한 설명)                                     |
| **`runtime_state.md`**   | - **실시간 상태 관리** (WebSocket, IndexedDB)                          | TypeScript `type`로 상태 정의 (구현 시 바로 사용)                                    |
| **`llm_adapters.md`**     | - **LLM 어댑터** (OpenAI, Bedrock, Google AI)                            | 각 LLM별 API 호출 함수 (재사용 코드)                                               |

---

### **추가적인 문서 (구현 중 보완)**
| **파일명**               | **내용**                                                                 | **공수 절약 방법**                                                                 |
|---------------------------|--------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| **`ui_template.md`**      | - **UI 템플릿** (와이어프레임 수준)                                     | Figma 디자인 + React/Vue.js 컴포넌트 스펙 (재사용)                                   |
| **`error_handling.md`**   | - **에러 처리 로직** (동기화, LLM 호출)                                 | 예외 처리 코드 예시 (구현 시 복사)                                                  |
| **`test_scenarios.md`**   | - **테스트 케이스** (단위 + E2E)                                           | Jest + Cypress 예시 코드 (구현 후 작성)                                              |

---

## **2. 공수 최적화 전략 (70% 설계 + 30% 구현)**
### **2.1. 설계 단계 (구현 전 90% 필수)**
1. **`architecture.md`**
   - **시스템 아키텍처** (Chrome Extension + Web UI + DB)
   - **TypeScript `interface`**로 DB/API 스펙 통합 (예시 코드 포함)
   ```typescript
   // types.ts (재사용 코드)
   interface RoundData {
     id: string;
     type: 'R1' | 'R2' | 'R3';
     llmAnswers: Record<string, string>;
     reviews: Array<{ llm: string; content: string }>;
   }
   ```

2. **`protocol.md`**
   - **LLM 협업 프로토콜** (R1/R2/R3 로직)
   - **상태전환 다이어그램** (간결한 설명)
   ```mermaid
   graph TD
     A[R1: Blind Response] --> B[R2: Peer Review]
     B --> C[R3: Final Decision]
   ```

3. **`runtime_state.md`**
   - **WebSocket/Polling** (동기화 관리)
   - **IndexedDB 상태 관리** (TypeScript `type`로 정의)
   ```typescript
   type SessionState = {
     syncTimeout: number;
     rounds: Array<RoundData>;
   };
   ```

---

### **2.2. 구현 단계 (30% 보완)**
1. **`ui_template.md`**
   - **Figma 디자인 + React/Vue.js 컴포넌트 스펙**
   - **예시 코드** (재사용)
   ```jsx
   // components/AnswerInput.jsx
   export default function AnswerInput({ llmType, onSubmit }) {
     return <input onChange={(e) => onSubmit(e.target.value)} />;
   }
   ```

2. **`error_handling.md`**
   - **에러 처리 로직** (예외 처리 코드)
   ```javascript
   // utils/errorHandler.js
   export const handleLLMError = (error) => {
     console.error("LLM 호출 실패:", error);
     showErrorMessage("LLM 호출 실패");
   };
   ```

3. **`test_scenarios.md`**
   - **테스트 케이스** (Jest + Cypress)
   ```javascript
   // test/llm.test.js
   test("LLM 호출 성공", async () => {
     const mockFetch = jest.fn();
     mockFetch.mockResolvedValueOnce({ json: () => ({ choices: [{ text: "테스트" }] }) });
     global.fetch = mockFetch;
     const result = await callLLM("test");
     expect(result).toBe("테스트");
   });
   ```

---

## **3. 재사용성 향상 (미래 프로젝트)**
### **3.1. 템플릿화 (재사용 가능)**
| **파일명**               | **재사용 방법**                                                                 |
|---------------------------|--------------------------------------------------------------------------------|
| **`requirements.md`**     | 다음 프로젝트에서 요구사항 정의로 재사용 (기본값 채우기)                  |
| **`architecture.md`**    | TypeScript `interface`로 재사용 (DB/API 스펙 재사용)                          |
| **`protocol.md`**         | 의사코드 + 상태전환 다이어그램 재사용 (로직 재사용)                         |
| **`ui_template.md`**      | React/Vue.js 컴포넌트 재사용 (UI 템플릿 재사용)                              |

---

## **4. 최종 공수 계산 (예시)**
| **단계**               | **공수**       | **절약 방법**                                                                 |
|-------------------------|----------------|------------------------------------------------------------------------------|
| 요구사항 정의          | 10h            | Markdown + GitHub Wiki (편집 속도 ↑)                                        |
| 설계서 작성            | 15h            | TypeScript `interface` + 의사코드 (재사용 코드)                           |
| 코드 구현              | 40h            | 재사용 API + Hooks (공수 ↓)                                                   |
| 테스트                | 20h            | Jest + Cypress (자동화 테스트)                                               |
| 배포                  | 10h            | GitHub Actions + CI/CD (자동 배포)                                            |
| **총 공수**: **95h → 85h (10% 절약)** |

---

## **5. No-Question Build를 위한 최종 가이드**
### **5.1. 설계 단계 (구현 전 90% 필수)**
1. **`architecture.md`**
   - **TypeScript `interface`**로 DB/API 스펙 통합 (코드 + 문서)
   - **시스템 아키텍처** (Chrome Extension + Web UI + DB)

2. **`protocol.md`**
   - **LLM 협업 프로토콜** (R1/R2/R3 로직)
   - **상태전환 다이어그램** (간결한 설명)

3. **`runtime_state.md`**
   - **WebSocket/Polling** (동기화 관리)
   - **IndexedDB 상태 관리** (TypeScript `type`로 정의)

### **5.2. 구현 단계 (30% 보완)**
1. **`ui_template.md`**
   - **Figma 디자인 + React/Vue.js 컴포넌트 스펙**
   - **예시 코드** (재사용)

2. **`error_handling.md`**
   - **에러 처리 로직** (예외 처리 코드)

3. **`test_scenarios.md`**
   - **테스트 케이스** (Jest + Cypress)

---

## **6. 결론**
✅ **No-Question Build를 위한 최적화된 설계 문서 구조**를 제시했습니다.
✅ **공수 절감**을 위해 **재사용 가능한 코드 + 문서**를 동일 구조로 유지합니다.
✅ **미래 프로젝트 재사용성**을 위해 **템플릿화**합니다.

**다음 단계를 진행해보세요:**

1. **`architecture.md`** 작성 (TypeScript `interface`로 DB/API 스펙 통합)
2. **`protocol.md`** 작성 (LLM 협업 프로토콜 + 상태전환 다이어그램)
3. **`runtime_state.md`** 작성 (WebSocket/Polling + IndexedDB 상태 관리)

이후 **`ui_template.md`**와 **`error_handling.md`**를 구현 중 보완합니다. 🚀