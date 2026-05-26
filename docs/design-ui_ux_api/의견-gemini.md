ChatGPT와 Claude의 의견, 그리고 제 이전 제안까지 모두 모아 분석해 보니 아주 명확한 **최적의 절충점**이 보입니다. Deeps님이 원하시는 **'향후 프로젝트 템플릿 자산화'**와 **'현실적인 개발 공수 밸런스'**를 모두 잡는 마스터 플랜입니다.

ChatGPT의 통찰처럼 **"설계 문서 자체보다 설계 규칙과 구조의 템플릿화"**가 핵심이며, Claude의 지적대로 **"사용자의 검토 피로도를 줄이는 10종 문서 최적화"**가 맞습니다. 그리고 제가 제안했던 **"Code-First(설계가 곧 소스 코드가 되는 구조)"**를 결합하면 완벽합니다.

줄글로 된 장황한 문서는 과감히 배제하고, 향후 다른 프로젝트에도 그대로 복사해서 폴더 구조로 쓸 수 있는 **[자산화 표준 설계 템플릿 및 코어 설계]**를 선언합니다.

---

## 🛠️ [최종 결정] 10대 설계 산출물 폴더 구조 및 Depth 정의

앞으로 구축할 모든 프로젝트의 `docs/` 폴더에 들어갈 표준 템플릿 구조입니다. 이번 프로젝트에서 이 구조를 채워나가며 자산화합니다.

```text
docs/
├── 01_requirements/        # [완료] 요구사항 정의서 v2.0 (Freeze)
├── 02_architecture/        # [★매우높음] 시스템 토폴로지 및 V3 하이브리드 흐름
├── 03_protocol/            # [★매우높음] UI ↔ Background ↔ Content 통신 (types.ts 호환)
├── 04_runtime_state/       # [★매우높음] 3단계 라운드 동기화 큐 및 타임아웃 상태 머신
├── 05_storage/             # [중간] IndexedDB 스키마 및 자동저장 규칙
├── 06_llm_adapter/         # [높음] 추상화 어댑터 인터페이스 규격 (DOM 셀렉터 분리)
├── 07_uiux/                # [중간] Single-Page Multi-Column 레이아웃 및 3대 상태 뷰
├── 08_build_convention/    # [낮음] 디렉토리 구조 및 Manifest V3 빌드 규칙
├── 09_md_format/           # [낮음] 익스포트용 Markdown 파일 스펙 컴포넌트
└── 10_error_handling/      # [높음] LLM 미응답/네트워크 끊김 예외 처리 및 체크리스트
```

---

## 🚀 No-Question & 공수 최소화를 위한 핵심 코어 설계 연쇄 작성

문서 양식 채우기 공수를 최소화하기 위해, 구현 단계에서 무조건 질문이 나올 수밖에 없는 **`03_protocol`**, `04_runtime_state`, `05_storage`를 하나의 **TypeScript 기반 인터페이스 및 상태 머신 의사코드(Pseudo-code)**로 통합 설계하여 제공합니다. 개발자는 이 코드를 그대로 복사해서 파일로 생성하면 설계 검토와 동시에 코딩이 시작됩니다.

### 📄 [03_protocol] + [05_storage] 데이터 및 통신 규격 (`types.ts`)

```typescript
/**
 * 멀티 LLM 협업 시스템 Core Type Definitions
 * 위치: src/types/index.ts (대시보드 및 확장 프로그램 공용)
 */

// 1. 시스템 지원 LLM 종류 정의
export type LLMType = 'CHATGPT' | 'CLAUDE' | 'GEMINI';

// 2. LLM별 라운드 진행 상태 코드
export type LLMStatus = 'IDLE' | 'LOADING' | 'SUCCESS' | 'TIMEOUT' | 'ERROR';

// 3. IndexedDB 스키마 구조 정의
export interface SessionRow {
  id?: number;              // Auto Increment ID
  title: string;            // 첫 번째 프롬프트 기반 자동 생성 타이틀
  activeLlmList: LLMType[]; // 사용자가 선택한 LLM 리스트 (최소 2개)
  createdAt: number;        // 타임스탬프
}

export interface RoundLogRow {
  id?: number;
  sessionId: number;        // Session.id와 매핑 (Index)
  roundNumber: 1 | 2 | 3;   // 최대 3라운드 제한
  llmType: LLMType;
  promptRaw: string;        // 해당 라운드에 주입된 프롬프트 원문
  responseRaw: string;      // 수집된 응답 원문 (TIMEOUT 시 '[TIMEOUT_NO_RESPONSE]')
  timestamp: number;
}

// 4. Chrome Extension Message Passing 프로토콜 규격
export type MessageAction = 
  | 'START_ROUND'       // 대시보드 -> Background: 라운드 시작 질의 요청
  | 'STREAM_RESPONSE'   // Content -> Background -> 대시보드: 실시간 텍스트 전달
  | 'ROUND_COMPLETE'    // Content -> Background -> 대시보드: 단일 LLM 최종 응답 완료
  | 'ABORT_ROUND';      // 대시보드 -> Background: 사용자 조기 종료/중단 개입

export interface ChromeExtensionMessage {
  action: MessageAction;
  payload: {
    sessionId: number;
    roundNumber: 1 | 2 | 3;
    targetLlm?: LLMType;
    promptText?: string;
    responseText?: string;
    errorMsg?: string;
  };
}
```

---

### 📄 [04_runtime_state] 3단계 라운드 동기화 의사코드 (`orchestrator.ts`)

개발자가 동기화 큐(Queue)와 5분 타임아웃을 질문 없이 구현할 수 있도록 핵심 오케스트레이션 로직을 명세합니다.

```typescript
/**
 * 3단계 라운드 동기화 오케스트레이터 의사코드 (Background Service Worker 내 구동)
 */
class RoundOrchestrator {
  private currentTimeoutId: NodeJS.Timeout | null = null;
  private pendingLLMs: Set<LLMType> = new Set();

  async executeRound(sessionId: number, roundNumber: number, activeLLMs: LLMType[], prompt: string) {
    this.pendingLLMs = new Set(activeLLMs);
    
    // 1. 5분 강제 타임아웃 타이머 가동 (REQ-09 보장)
    this.startGlobalTimeout(sessionId, roundNumber);

    // 2. 각 LLM 확장 프로그램 Content Script로 프롬프트 명령 분기 던지기
    for (const llm of activeLLMs) {
      this.dispatchToContentScript(llm, {
        action: 'START_ROUND',
        payload: { sessionId, roundNumber, targetLlm: llm, promptText: prompt }
      });
    }
  }

  // Content Script로부터 응답을 수신했을 때 호출되는 브로커 콜백
  onLlmResponseReceived(llm: LLMType, responseText: string, sessionId: number, roundNumber: number) {
    if (!this.pendingLLMs.has(llm)) return; // 이미 타임아웃 처리된 경우 무시

    // 1. 대시보드 UI로 완료 상태 메시지 포워딩
    this.sendToDashboard({ action: 'ROUND_COMPLETE', payload: { sessionId, roundNumber, targetLlm: llm, responseText } });
    
    // 2. 대기 큐에서 제거
    this.pendingLLMs.delete(llm);

    // 3. 동기화 검증: 모든 LLM이 응답 완료했는지 체크 (REQ-08 보장)
    if (this.pendingLLMs.size === 0) {
      this.clearGlobalTimeout();
      this.finalizeRound(sessionId, roundNumber);
    }
  }

  private startGlobalTimeout(sessionId: number, roundNumber: number) {
    this.currentTimeoutId = setTimeout(() => {
      // 5분 도래 시 아직 대기 중인 모든 LLM 강제 타임아웃 처리
      for (const failedLlm of this.pendingLLMs) {
        this.sendToDashboard({ 
          action: 'ROUND_COMPLETE', 
          payload: { sessionId, roundNumber, targetLlm: failedLlm, responseText: '[TIMEOUT_NO_RESPONSE]' } 
        });
      }
      this.pendingLLMs.clear();
      this.finalizeRound(sessionId, roundNumber);
    }, 5 * 60 * 1000); // 5분
  }

  private finalizeRound(sessionId: number, roundNumber: number) {
    // 대시보드에게 IndexedDB 자동 저장 프로세스를 트리거하라고 알림 (REQ-18 보장)
    this.sendToDashboard({ action: 'ROUND_COMPLETE', payload: { sessionId, roundNumber } });
  }
}
```

---

## 🏁 다음 진행을 위한 체크리스트 확인

가장 뼈대가 되고 재사용성이 높은 핵심 설계 데이터 규격(Protocol, Storage, Runtime State)이 코드 레벨로 도출되었습니다. 

이 설계 규칙과 소스코드가 Deeps님이 원하시던 **"No-Question 빌드가 가능하면서 공수가 최소화된 품질의 밸런스"**에 부합하는지 확인해 주세요. 

이 방향이 맞다면, 다음으로 기술 제약이 가장 심해 유연한 어댑터 구조 설계가 필요한 **`06_llm_adapter (DOM 주입 및 추출 규칙)`**에 대한 코드 스펙 설계를 진행하겠습니다. 사인을 부탁드립니다!