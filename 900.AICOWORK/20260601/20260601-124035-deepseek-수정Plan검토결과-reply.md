# DeepSeek — Opus 수정Plan검토결과 재검토 및 Plan v3

> **생성일**: 2026-06-01 12:40
> **대상 문서**: `20260601-123624-claude-수정Plan검토결과.md` (Claude Opus 4.8)
> **선행 문서**: `20260601-121916-구현검토결과.md` (C1/M1/M2 원검토 / Opus)
> **선행 문서**: `20260601-123721-deepseek-구현전-검토결과.md` (Plan v2 / DeepSeek)
> **검토 주체**: OpenCode (DeepSeek v4 Flash Free)
> **목적**: Opus 수정Plan검토 결과를 코드 레벨에서 재검증하고 최종 Plan v3 수립

---

## 0. 종합 판정

| Opus 코멘트                           | 재검토 결과                                    | Plan v3 반영     |
| ------------------------------------- | ---------------------------------------------- | ---------------- |
| C1 dom-adapter.ts stopObserve hoist   | ✅ 동의                                        | 반영             |
| C1 dom-adapter-bot.ts abort fallback  | ✅ 동의                                        | 반영             |
| double-DONE 가드                      | ❌ 실무 영향 0 — port.disconnect()로 차단됨    | 미반영           |
| **bg cleanup DONE/ERROR 통보 (신규)** | **✅ CRITICAL — CS 탭 종료 시 별도 hang 경로** | **신규 반영**    |
| M1 재시도 직렬화                      | ✅ 동의                                        | 반영             |
| M2 proxy-fetch.ts 삭제                | ✅ 동의                                        | 반영             |
| Happy path 회귀 검증                  | ✅ 동의                                        | 검증 절차에 반영 |

---

## 1. double-DONE 가드 검증 (Opus 🟡 제안 → ❌ 불필요)

### Opus 주장

observeResponse 자연 done과 STOP이 거의 동시에 발생하면 STOP 핸들러가 DONE을 한 번 더 보내 `controller.close()` 재호출 가능.

### 코드 레벨 추적

```
1차 DONE (natural done):
  CS → port.postMessage({action:'DONE'})
  → bg relay → app port.onMessage
  → params.onEvent({type:'DONE'})    // controller.close()
  → cleanup() → port.disconnect()    // ← app port 차단

2차 DONE (STOP handler):
  CS → port.postMessage({action:'DONE'})
  → bg relay
  → app port: 이미 disconnect 상태
  → Chrome runtime: 메시지 폐기 (onMessage 미도달)
```

**`port.disconnect()`는 동기(synchronous)로 작동** — Chrome은 disconnect된 port로의 메시지 전달을 즉시 중단함. 따라서 2차 DONE은 app의 `port.onMessage` 리스너에 도달하기 전에 차단됨.

### 판정

- 이론적 경합: 존재
- 실무 영향: **0** (controller.close() 재호출 발생 안 함)
- 반영: **불필요**

---

## 2. Background cleanup DONE 통보 (Opus 🟠 신규 발견 → ✅ CRITICAL)

### 문제 재현

C1은 CS 레벨에서 STOP 시 DONE 미통보가 원인이었으나, **CS 탭 자체가 사라지는 경우는 전혀 다른 경로**이며 별도 버그.

```
CS 탭 종료 (사용자가 LLM 탭을 닫음 / 탭 navigated away)
  → csPort.onDisconnect 자동 발생
  → background cleanup():
      appPort.onMessage.removeListener(stopListener)  // STOP forwarding만 제거
      csPort.disconnect()                              // 이미 dead
      // ← appPort에 DONE/ERROR 미통보!
  → appPort 생존
  → DomAdapterBot.doSendMessage()의 for await (gen) 영구 대기
  → useRoundEngine의 await Promise.all() hang
  → history.push / persistRoundSnapshot 미실행
  → UI는 WAITING 상태로 영구 대기
```

### 문제 위치

**`background/index.ts:112-119`**:

```typescript
const cleanup = () => {
  appPort.onMessage.removeListener(stopListener);
  try {
    csPort.disconnect();
  } catch {
    /* already gone */
  }
  // ← appPort에 아무런 종료 통보도 하지 않음
};
```

### 수정

```typescript
const cleanup = () => {
  appPort.onMessage.removeListener(stopListener);
  try {
    csPort.disconnect();
  } catch {
    /* already gone */
  }
  try {
    appPort.postMessage({
      action: "ERROR",
      code: "ERR_002",
      message: "LLM tab connection lost",
    });
  } catch {}
};
```

**3줄 추가**. appPort가 이미 disconnect된 경우 `postMessage`가 throw → `catch`에서 무시.

### C1과의 관계

| 경로      | 발생 조건                             | 버그                                       | 수정 위치                     |
| --------- | ------------------------------------- | ------------------------------------------ | ----------------------------- |
| C1 경로   | 사용자 STOP + deepseek(stopButton 無) | observeResponse timer 미중단 + DONE 미통보 | `dom-adapter.ts:113-117`      |
| 신규 경로 | CS 탭 종료/사망 (STOP과 무관)         | cleanup이 appPort에 종료 미통보            | `background/index.ts:112-119` |

두 경로는 **독립적**이며, 모두 수정해야 모든 hang 상황이 해소됨.

---

## 3. 최종 수정 Plan v3

### 수정 대상 (4개 파일, ~15줄)

| 우선순위 | 파일                             | 수정 내용                                         | 예상 공수 |
| -------- | -------------------------------- | ------------------------------------------------- | --------- |
| **P0**   | `content-script/dom-adapter.ts`  | stopObserve hoist + STOP서 timer 중단 + DONE 통보 | ~5줄      |
| **P0**   | `app/bots/dom-adapter-bot.ts`    | abort catch port-dead 시 직접 DONE + cleanup      | ~2줄      |
| **P0**   | **`background/index.ts` (신규)** | **cleanup에서 appPort에 ERROR 통보**              | **~3줄**  |
| P1       | `background/index.ts` (기존)     | M1: connectToCSWithRetry 재시도 직렬화            | ~5줄      |
| P2       | `services/proxy-fetch.ts`        | 파일 삭제                                         | 1줄       |

### 미변경 사항

- double-DONE 가드: 불필요 (section 1 증명)
- L1~L4: 의도적 설계 또는 저위험으로 유지
- abstract-bot.ts: ReadableStream 로직 정상
- manifest.config.ts: 이미 222 명세 충족

### 영향/회귀 분석

| 변경                      | 회귀 위험 | 사유                                    |
| ------------------------- | --------- | --------------------------------------- |
| dom-adapter.ts C1         | 無        | 기존 동작의 상위집합 — STOP 처리만 보강 |
| dom-adapter-bot.ts C1     | 無        | catch 경로만 신규 — 정상 경로 영향 0    |
| background cleanup (신규) | 無        | cleanup 내 3줄 추가 — 기존 흐름 변경 無 |
| background M1             | 無        | 재시도 로직만 변경 — 성공 시 동일       |
| proxy-fetch.ts 삭제       | 無        | import 0건 — grep 확인 완료             |

### 검증 절차

```
1. 정적 검증:
   npx tsc --noEmit        → 0 errors
   yarn build              → pass
   grep "proxyFetch\|proxy-fetch" src/  → 0 (M2)

2. E2E 수동 검증:
   IT-01: 정상 3라운드 전송 → 모든 LLM 응답 + DONE (회귀 확인)
   IT-02: deepseek 포함 라운드 → 생성 중 STOP → STOPPED + history 저장 (C1)
   IT-03: CS 탭 종료 (LLM 탭 닫기) → hang 없이 ERROR 복구 (신규)
   IT-04: 미로그인 LLM 전송 → ERR_002 오류 (거짓 양성 없음, M1)
   IT-05: DONE 후 저장 → chatgpt.md 등 MD 다운로드
```

---

## 4. 전체 문서 위치

| 문서                          | 파일명                                                   | 작성자                 |
| ----------------------------- | -------------------------------------------------------- | ---------------------- |
| 1차 구현검토결과              | `20260601-121916-구현검토결과.md`                        | Opus 4.8               |
| DeepSeek Plan v2              | `20260601-123721-deepseek-구현전-검토결과.md`            | DeepSeek               |
| Opus Plan검토결과             | `20260601-123624-claude-수정Plan검토결과.md`             | Opus 4.8               |
| **DeepSeek 재검토 + Plan v3** | **`20260601-124035-deepseek-수정Plan검토결과-reply.md`** | **DeepSeek (본 문서)** |

---

_본 문서는 Opus 수정Plan검토결과를 DeepSeek이 코드 레벨에서 재검증하고 최종 Plan v3를 수립한 결과물입니다._
_코드 수정은 별도 요청 시 진행합니다._
