# LLM Adapter 명세

> ⚠️ **대체됨 — 구현·검증 기준은 `210.설계산출물-구현적용` + `100.START/VER-002` (2026-05-29).** 본 200-초안은 출처·대조용. 수치/구조 차이는 210 우선.

> 변경 이력: v1=원본, v2=2026-05-28 설계 추가건

## 공통 인터페이스

```javascript
class LLMAdapter {
  // prompt 전송 (실패 시 throw → UI에 "로그인 필요" 표시)
  async sendPrompt(prompt: string, context: object): void

  // 응답 추출 (DOM polling)
  async extractResponse(timeout?: number): Promise<string>

  // 응답 대기 상태 확인
  async isResponding(): boolean

  // STOP 신호 전달 (DOM에 stop 버튼 클릭 등)
  async stop(): void
}
```

## LLM별 상세

### 1. ChatGPT (chatgpt.com)

| 항목 | 내용 |
|------|------|
| URL | https://chatgpt.com |
| prompt 주입 | `#prompt-textarea` textarea에 값 설정 + Enter keydown |
| 전송 버튼 | `button[data-testid="send-button"]` click |
| 응답 영역 | `article` 요소 내 마지막 `div.markdown` |
| 응답 완료 감지 | `button[data-testid="stop-button"]` 존재 여부 (생성 중이면 존재) |
| STOP | `button[data-testid="stop-button"]` click |

### 2. Claude (claude.ai)

| 항목 | 내용 |
|------|------|
| URL | https://claude.ai |
| prompt 주입 | `div[contenteditable="true"]`에 text 설정 + Enter |
| 전송 버튼 | `button[aria-label*="Send"]` click |
| 응답 영역 | `div.font-claude-message` 마지막 요소 |
| 응답 완료 감지 | 더 이상 `div.animate-pulse` 없음 |
| STOP | `button[aria-label*="Stop"]` click |

### 3. Gemini (gemini.google.com)

| 항목 | 내용 |
|------|------|
| URL | https://gemini.google.com |
| prompt 주입 | `div.ql-editor`에 text + Enter |
| 전송 버튼 | `button.send-button` click |
| 응답 영역 | `div.response-content` 마지막 |
| 응답 완료 감지 | `div.thinking` 또는 스피너 제거 |
| STOP | `button.stop-button` click |

### 4. DeepSeek (chat.deepseek.com)

| 항목 | 내용 |
|------|------|
| URL | https://chat.deepseek.com |
| prompt 주입 | `textarea`에 text + Enter |
| 응답 영역 | `div.markdown-body` 마지막 |
| STOP | `button[aria-label="Stop"]` click |

### 5. Grok (grok.com)

| 항목 | 내용 |
|------|------|
| URL | https://grok.com |
| prompt 주입 | `textarea`에 text + Enter |
| 응답 영역 | `div.message-content` 마지막 |
| STOP | `button.stop-generation` click |

### 6. Perplexity (perplexity.ai)

| 항목 | 내용 |
|------|------|
| URL | https://www.perplexity.ai |
| prompt 주입 | `textarea` 또는 input 필드 |
| 응답 영역 | `div.prose` 마지막 |
| STOP | `button:has(svg.lucide-stop)` click |

## DOM 변경 대응 전략 (미해결)

- **일괄 대응**: 6종 [v2] LLM selector를 JSON config로 관리
- **버저닝**: 사이트 업데이트 시 selector 버전 증가
- **fallback**: selector 실패 시 사용자에게 수동 입력 유도

```yaml
# adapter-config.yaml 예시
adapters:
  chatgpt:
    version: 1
    selectors:
      promptArea: "#prompt-textarea"
      sendButton: "button[data-testid='send-button']"
      responseArea: "article > div.markdown"
      stopButton: "button[data-testid='stop-button']"
      # [v2] sessionCookie 제거 — prompt 실패 시 "로그인 필요"
```
