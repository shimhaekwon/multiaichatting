# 216. 교차전달 · context 빌더 (self-exclusion + YAML)

> **자기완결**. 구현: `src/app/utils/context-builder.ts`(신규). 의존성: `yaml`(추가). 토큰 근사: `gpt3-tokenizer`(기존 의존성). [대조용 초안: 200-초안/220§3·240]

## 1. 목적

라운드 종료 후 **각 LLM에 "자기 응답을 제외한 타 LLM 응답만"** 을 YAML로 직렬화해 다음 라운드 prompt에 주입한다. → echo chamber(자기강화) 방지, 독립적 사고 유도. **핵심**: 기존 `bot.sendMessage({prompt})`에 합성된 prompt를 넘기는 것이므로 봇 코드는 그대로다.

## 2. 타입 & 함수

```ts
import { stringify as yamlStringify } from 'yaml'   // [H2] named import — esModuleInterop:false에서 default-interop 회피
import GPT3Tokenizer from 'gpt3-tokenizer'
import { BotId } from '~app/bots'
import { RoundRecord } from '~types/round'

export interface LlmContext {
  roundNo: number
  userPrompt: string
  others: Record<string, string>   // key=botId (고정), value=응답 내용
}

/** self-exclusion: target 제외, status==='completed'만. [M-3] 1라운드는 VER-002 14-1대로 "타 LLM 키만, 값 빈 문자열" */
export function buildContextForLlm(
  targetLlm: BotId,
  prevRound: RoundRecord | undefined,
  nextRoundNo: number,
  userPrompt: string,
  participants: BotId[],                               // [M-3] 현재 라운드 참여 슬롯(자기 포함) — 1라운드 빈 구조 키 생성용
): LlmContext | null {
  const others: Record<string, string> = {}
  if (prevRound) {
    for (const [llm, resp] of Object.entries(prevRound.responses)) {
      if (llm !== targetLlm && resp && resp.status === 'completed' && resp.content) {
        others[llm] = resp.content                     // ← 자기 자신 제외, 완료분만 (2라운드+)
      }
    }
  } else {
    // [ver.002 14-1 — 리터럴 준수] 1라운드: 타 LLM 키만 존재, 값은 빈 문자열("LLM만 있고 비어있음")
    for (const llm of participants) if (llm !== targetLlm) others[llm] = ''
  }
  if (Object.keys(others).length === 0) return null    // 참여 1종(타 LLM 없음)이면 블록 생략
  return { roundNo: nextRoundNo, userPrompt, others }
}

/** YAML 직렬화 — 토큰 최소화 + 시인성. others만 직렬화(라벨 모호성 제거) */
export function serializeOthersYaml(others: Record<string, string>): string {
  return yamlStringify(others)         // 예) "claude: 경쟁사 분석부터...\ngemini: 소비자 인터뷰를..."
}

/** 토큰 제한 지시문 (FR-02) — 매 라운드 request에 자동 부가 */
export function tokenLimitInstruction(tokenLimit: number): string {
  return `\n\n(응답은 약 ${tokenLimit} 토큰 이내로 간결하게 작성해 주세요.)`
}

/** 최종 prompt 합성 */
export function composePrompt(userPrompt: string, ctx: LlmContext | null, tokenLimit: number): string {
  let p = userPrompt
  if (ctx && Object.keys(ctx.others).length) {
    p += `\n\n[이전 라운드 다른 AI 의견]\n` + serializeOthersYaml(ctx.others)
  }
  p += tokenLimitInstruction(tokenLimit)
  return p
}

// [M-4] gpt3-tokenizer는 CJS. tsconfig esModuleInterop:false 라 default import 상호운용 주의.
//   런타임에 .default 보정 필요 시: const Ctor:any = (GPT3Tokenizer as any).default ?? GPT3Tokenizer
//   생성자가 큰 BPE 사전을 로드 → 지연 init + 전체 try/catch(생성 실패도 폴백). 카운트는 표시용이라 실패해도 무해.
let _tk: GPT3Tokenizer | null = null
export function approxTokens(text: string): number {
  if (!text) return 0
  try {
    if (!_tk) _tk = new GPT3Tokenizer({ type: 'gpt3' })   // 지연 init(모듈 로드/생성 실패도 여기서 폴백)
    return _tk.encode(text).bpe.length
  } catch { return Math.ceil(text.length / 2) }            // 근사(표시용)
}
```

## 3. 합성 prompt 예시 (Round 2, ChatGPT 대상)

```
시장 조사 아이디어를 내줘

[이전 라운드 다른 AI 의견]
claude: 경쟁사 분석부터 시작하는 게 효율적입니다
gemini: 소비자 인터뷰를 병행하는 것이 좋습니다

(응답은 약 1000 토큰 이내로 간결하게 작성해 주세요.)
```

→ ChatGPT 자기 응답은 **포함되지 않음**(self-exclusion). 각 LLM은 자기 `conversationId`로 맥락을 유지하므로, 주입은 "타 AI 의견"만으로 최소화(`maxRounds`/`tokenLimit` 등 메타는 제외).

## 4. 주의

- **key 규칙 [G8 확정]**: `others`의 key = **botId**(소문자, 예 `chatgpt:`)로 **고정**(VER-002 항목3 예시와 일치). 표시명 치환은 하지 않음(일관성·토큰 절약).
- **중단된 LLM 제외**: STOPPED 라운드에서 미수신/`error` 응답은 `status!=='completed'`라 자동 제외(self-exclusion).
- **토큰 카운트는 근사**(브라우저·LLM별 토크나이저 상이). 표시/로깅용이며 하드 컷이 아님 — 실제 제한은 지시문으로 LLM에 위임.
- YAML 직렬화는 멀티라인 응답 시 `|` 블록 스칼라로 자동 처리됨(`yaml` 라이브러리).
- **[ver.002 14-1/14-2 — M-3 확정]** 교차전달은 **1라운드부터** 적용된다. 1라운드는 `buildContextForLlm`가 **타 LLM 키만 빈 값으로** 채워 `[이전 라운드 다른 AI 의견]` 블록에 참여자 명단(값 없음)이 포함된다 — VER-002 14-1 "1round엔 LLM만 있고 비어있음" **리터럴 준수**(기존 "생략 권장"에서 변경). 토큰 절약을 원하면 `participants`에 빈 배열을 넘겨 1라운드 블록을 생략하는 opt-out 가능(기본=리터럴 준수). LLM에 전달하는 데이터(`others`)만 **context**로 간주(메타 제외).
