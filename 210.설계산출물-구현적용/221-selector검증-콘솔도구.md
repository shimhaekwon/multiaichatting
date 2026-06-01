# 221. selector 검증 콘솔 도구 (Phase 2 사전 검증)

> **구현 전 검증용**. multichat 소스를 수정하지 않고, **대상 LLM 사이트의 DevTools 콘솔에 붙여넣어** ① selector 유효성 ② 타이핑→전송→응답 읽기(=DOM 어댑터 접근)가 실제로 되는지 사용자가 직접 확인하는 도구. 여기서 확정한 selector를 `adapter-config.json`(217 §3.2)에 반영하면 됨.
> ⚠️ `chatHubRoundtrip()`은 **실제로 LLM에 메시지를 보냅니다**(quota 소모·대화기록 남음). selector 점검만 할 땐 `chatHubProbe()`만.

## 1. 사용법
1. 대상 사이트(예: `chatgpt.com`)에 **로그인**한 상태로 채팅 화면을 연다.
2. **F12 → Console 탭**.
3. 아래 **§2 스니펫 전체**를 붙여넣고 Enter (함수 등록됨).
4. **§3에서 해당 사이트 CFG**를 복사해 붙여넣고 Enter (선택자 세팅).
5. `chatHubProbe()` 실행 → 표로 각 selector 매칭 확인. 빨간 ⚠ 나오면 selector 수정.
6. (선택) `chatHubRoundtrip('안녕, 한 줄 자기소개')` → 실제 1회 왕복 + done 판정 관찰.
7. 잘 되는 selector를 `adapter-config.json`의 해당 키에 기록(version=1).

## 2. 콘솔 스니펫 (공통, 1회 붙여넣기)

```js
/* === ChatHub DOM 어댑터 selector 검증 도구 (구현 전 PoC) === */
window.CFG = window.CFG || {};
function $1(sel){ return sel ? document.querySelector(sel) : null; }
function $A(sel){ return sel ? [...document.querySelectorAll(sel)] : []; }

window.chatHubProbe = function(){
  const C = window.CFG;
  const rows = [['promptArea',C.promptArea],['sendButton',C.sendButton],
    ['responseArea',C.responseArea],['stopButton',C.stopButton],['generatingFlag',C.generatingFlag]]
    .map(([k,sel])=>{
      const els=$A(sel), last=els[els.length-1];
      return {key:k, selector:sel||'(none)', found:els.length,
        sample:last?((last.innerText||last.value||'').slice(0,40).replace(/\s+/g,' ')):''};
    });
  console.table(rows);
  if(!$A(C.promptArea).length) console.warn('⚠ promptArea 미발견 → selector 수정 필요');
  if(!$A(C.responseArea).length) console.warn('⚠ responseArea 미발견 → selector 수정 필요');
  return rows;
};

window.chatHubRoundtrip = async function(prompt='안녕, 한 줄로 자기소개 해줘'){
  const C = window.CFG, box=$1(C.promptArea);
  if(!box){ console.error('promptArea 없음 — CFG 확인'); return; }
  box.focus();
  if(C.promptIsContentEditable){
    box.textContent = prompt;
    box.dispatchEvent(new InputEvent('input',{bubbles:true,inputType:'insertText',data:prompt}));
    // 안 먹으면 폴백: document.execCommand('insertText',false,prompt)
  } else {
    const set=Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype,'value').set;
    set.call(box,prompt); box.dispatchEvent(new Event('input',{bubbles:true}));
  }
  await new Promise(r=>setTimeout(r,300));
  const btn=$1(C.sendButton);
  if(btn) btn.click();
  else box.dispatchEvent(new KeyboardEvent('keydown',{key:'Enter',code:'Enter',keyCode:13,which:13,bubbles:true}));
  console.log('%c전송함. 응답 관찰...','color:#4488FF');
  const start=Date.now(); let lastText='', lastChange=Date.now();
  return await new Promise(resolve=>{
    const t=setInterval(()=>{
      const els=$A(C.responseArea), cur=els.length?(els[els.length-1].innerText||''):'';
      if(cur!==lastText){ lastText=cur; lastChange=Date.now(); }
      const flag=C.generatingFlag?$1(C.generatingFlag):null, stop=C.stopButton?$1(C.stopButton):null;
      const el=Date.now()-start; let done=false, why='';
      if(C.generatingFlag){ if(!flag&&lastText){done=true;why='generatingFlag 사라짐';} }
      else if(C.stopButton){ if(!stop&&lastText){done=true;why='stopButton 사라짐';} }
      else { if(lastText&&Date.now()-lastChange>(C.doneDebounceMs||1500)){done=true;why='텍스트 안정화';} }
      if(el>120000){ done=true; why='타임아웃 120s(수동확인)'; }
      if(done){ clearInterval(t);
        console.log(`%c✅ done (${why}, ${(el/1000).toFixed(1)}s, ${lastText.length}자)`,'color:#44BB44;font-weight:bold');
        console.log('--- 응답 ---\n'+lastText);
        resolve({done:why,ms:el,text:lastText}); }
    },300);
  });
};
console.log('%c준비됨 → §3 CFG 붙여넣고 chatHubProbe() 먼저','color:#4488FF;font-weight:bold');
```

## 3. 6종 CFG 프리셋 (217 §3.2 추정치 — 사이트에서 검증·수정)

```js
// ── chatgpt.com ──
window.CFG={promptArea:'div#prompt-textarea',promptIsContentEditable:true,sendButton:'button[data-testid="send-button"]',responseArea:'div[data-message-author-role="assistant"]',stopButton:'button[data-testid="stop-button"]',generatingFlag:'',doneDebounceMs:1500};
// ── claude.ai ──
window.CFG={promptArea:'div.ProseMirror',promptIsContentEditable:true,sendButton:'',responseArea:'div.font-claude-message',stopButton:'button[aria-label="Stop response"]',generatingFlag:'',doneDebounceMs:1500};
// ── www.perplexity.ai ──
window.CFG={promptArea:'textarea',promptIsContentEditable:false,sendButton:'',responseArea:'div.prose',stopButton:'button[aria-label="Stop"]',generatingFlag:'',doneDebounceMs:1500};
// ── gemini.google.com ──
window.CFG={promptArea:'div.ql-editor',promptIsContentEditable:true,sendButton:'button.send-button',responseArea:'div.response-content',stopButton:'button.stop-button',generatingFlag:'div.thinking',doneDebounceMs:1500};
// ── chat.deepseek.com ──
window.CFG={promptArea:'textarea',promptIsContentEditable:false,sendButton:'',responseArea:'div.markdown-body',stopButton:'button[aria-label="Stop"]',generatingFlag:'',doneDebounceMs:1500};
// ── grok.com ──
window.CFG={promptArea:'textarea',promptIsContentEditable:false,sendButton:'',responseArea:'div.message-content',stopButton:'button.stop-generation',generatingFlag:'',doneDebounceMs:1500};
```

## 4. 결과 해석
- `chatHubProbe()` 표에서 **promptArea·responseArea의 found ≥ 1** 이면 핵심 selector OK. found=0이면 그 사이트에서 Elements 패널로 실제 요소 확인 후 selector 교체.
- `sendButton` found=0이면 빈 `''`로 두고 Enter 전송 경로 사용(스니펫이 자동).
- `chatHubRoundtrip()`이 **응답 텍스트를 끝까지 출력 + done 사유 표시**되면 그 사이트는 DOM 접근 가능 확정. done 사유가 "타임아웃"만 뜨면 done 판정 selector(generatingFlag/stopButton) 보강 필요.

## 5. 알려진 이슈 & 폴백
- **contenteditable 입력 안 먹힘**(ProseMirror/ql-editor 등): `box.focus()` 후 콘솔에서 `document.execCommand('insertText',false,'테스트')` 시도. 그래도 안 되면 `beforeinput`/`paste` 이벤트 디스패치 필요(에디터별 상이).
- **Trusted Types / CSP**: 일부 사이트(예: Google)는 콘솔 스크립트를 제한할 수 있음 → 그 경우 Elements 패널 수동 inspect로 selector만 확보.
- **responseArea가 여러 개**: 스니펫은 **마지막 요소**를 최신 응답으로 간주(설계와 동일). 사이트가 다르면 selector를 더 구체화.
- **스트리밍 중 DOM 교체**: 일부 사이트는 응답을 통째 교체 → debounce(1500ms)로 안정화 감지(조정 가능).

## 6. 반영
검증된 값 → `src/content-script/adapter-config.json`(217 §3.2)의 해당 키에 기록하고 `version:1` 확정. 이후 구현(Phase 2)에서 `dom-adapter.ts`가 이 config를 로드.

> 본 도구는 **구현 전 검증용**이며 multichat 소스가 아니다(제품 코드 아님). 실제 어댑터 구현·빌드는 사용자가 진행.

## 7. Phase 2 검증 결과 기록 (실측 확정 selector)

> 사이트에서 검증 완료된 값만 기록. 구현 시 `adapter-config.json`에 그대로 반영.

**🎉 Phase 2 완료 (2026-05-29) — 6종 핵심 selector 실측 확정.** 6종 전부 **입력·응답 확정**. 정지버튼: ChatGPT·Claude·Gemini·Grok·Perplexity 확보(Perplexity는 아이콘 기반 미검증), **DeepSeek만 best-effort**(div[role=button] → done=debounce). 입력 4종은 contenteditable(rich editor) → 입력 주입 폴백 필요(§5).

#### 통합 요약표 (6종)
| LLM | promptArea | sendButton | responseArea | stopButton |
|-----|-----------|-----------|--------------|-----------|
| chatgpt | `div#prompt-textarea`(CE) | `button[data-testid="send-button"]` | `div[data-message-author-role="assistant"]` | `button[data-testid="stop-button"]` |
| claude | `div.ProseMirror`(CE) | (Enter) | `div.standard-markdown` | `button[aria-label="응답 중단"]` |
| gemini | `div.ql-editor`(CE) | (Enter) | `div.response-content` | `button[aria-label="대답 생성 중지"]` |
| deepseek | `textarea` | (Enter) | `div.ds-assistant-message-main-content` | best-effort |
| perplexity | `#ask-input`(CE) | `button[aria-label="제출"]` | `div[id^="markdown-content"]` | `button:has(use[*\|href$="pplx-icon-player-stop-filled"])` |
| grok | `textarea` | `button[data-testid="chat-submit"]` | `div.response-content-markdown` | `button[aria-label="모델 응답 중지"]` |

#### 통합 `adapter-config.json` (구현 시 복사 — selector 내부는 작은따옴표 사용, 유효 CSS)
```json
{
  "chatgpt":    { "version": 1, "promptArea": "div#prompt-textarea", "promptIsContentEditable": true,  "sendButton": "button[data-testid='send-button']", "responseArea": "div[data-message-author-role='assistant']", "stopButton": "button[data-testid='stop-button']" },
  "claude":     { "version": 1, "promptArea": "div.ProseMirror",     "promptIsContentEditable": true,  "sendButton": "", "responseArea": "div.standard-markdown", "stopButton": "button[aria-label='응답 중단'], button[aria-label='Stop response']" },
  "gemini":     { "version": 1, "promptArea": "div.ql-editor",        "promptIsContentEditable": true,  "sendButton": "", "responseArea": "div.response-content", "stopButton": "button[aria-label='대답 생성 중지'], button:has(mat-icon[fonticon='stop'])" },
  "deepseek":   { "version": 1, "promptArea": "textarea",             "promptIsContentEditable": false, "sendButton": "", "responseArea": "div.ds-assistant-message-main-content", "stopButton": "", "doneDebounceMs": 1500 },
  "perplexity": { "version": 1, "promptArea": "#ask-input",           "promptIsContentEditable": true,  "sendButton": "button[aria-label='제출'], button[aria-label='Submit']", "responseArea": "div[id^='markdown-content']", "stopButton": "button:has(use[*|href$='pplx-icon-player-stop-filled'])" },
  "grok":       { "version": 1, "promptArea": "textarea",             "promptIsContentEditable": false, "sendButton": "button[data-testid='chat-submit']", "responseArea": "div.response-content-markdown", "stopButton": "button[aria-label='모델 응답 중지'], button[aria-label='Stop model response']" }
}
```
> ⚠️ aria-label 기반(claude/gemini/grok)은 본래 **한글 UI 기준**이었으나 **[M-2] 위 json에 영어 라벨/아이콘 fallback을 CSS `,` 그룹으로 반영**(claude=영어, gemini=`mat-icon[fonticon='stop']` 아이콘, grok=영어, perplexity=아이콘+Submit). 잔여: Gemini 전송(Enter 가정·확인), DeepSeek stop(div[role=button] composer-scoped) 보강은 구현 시.
>
> ⚠️ **[222] 위 "통합 adapter-config.json"이 정본(copy-verbatim)** — 그 아래 사이트별 블록은 검증 로그(참고). **perplexity `stopButton`의 `:has(use[*|href…])`는 `document.querySelector`에서 무효**(네임스페이스 선택자라 throw) → dom-adapter는 `document.querySelector("use[*|href$='pplx-icon-player-stop-filled']")?.closest('button')` 경로로 처리(222 §8 `findStop`). Gemini 전송(Enter)·DeepSeek stop은 222 §18 절차로 구현 시 확정.

---

### ✅ chatgpt.com — 4 selector 검증 완료 (2026-05-29)
```json
"chatgpt": { "version": 1, "promptArea": "div#prompt-textarea", "promptIsContentEditable": true, "sendButton": "button[data-testid=\"send-button\"]", "responseArea": "div[data-message-author-role=\"assistant\"]", "stopButton": "button[data-testid=\"stop-button\"]" }
```
- done 판정 = **stopButton 소멸**(generatingFlag 불필요). 전송 = sendButton 또는 Enter.
- 비고: 송신/정지 버튼은 동일 요소(`id="composer-submit-button"`)가 상태에 따라 `data-testid`를 `send-button`↔`stop-button`로 토글. 입력 비면 send-button 숨김(정상). 초안 추정치와 일치.

### ✅ claude.ai — 4종 전부 확정 (2026-05-29)
```json
"claude": { "version": 1, "promptArea": "div.ProseMirror", "promptIsContentEditable": true, "sendButton": "", "responseArea": "div.standard-markdown", "stopButton": "button[aria-label=\"응답 중단\"]", "generatingFlag": "" }
```
- promptArea `div.ProseMirror`(CE) ✅ / 전송 = Enter(sendButton 빈값) ✅ / responseArea **`div.standard-markdown`** ✅ (응답 본문 `<p>`=`font-claude-response-body`, 컨테이너로 전체 캡처) / stopButton **`button[aria-label="응답 중단"]`** ✅
- done 판정 = stopButton 소멸.
- ⚠️ stopButton은 **언어 종속**: 영어 UI면 `button[aria-label="Stop response"]`. 다국어 대응 권장 selector = `button[aria-label="응답 중단"], button[aria-label="Stop response"]`. (class `_fill_…`/`_secondary_…`는 해시라 사용 금지)
- 초안 `div.font-claude-message`·영어 `aria-label="Stop response"` → 한글 환경에서 폐기.
### ✅ www.perplexity.ai — 4종 전부 확정 (2026-05-29)
```json
"perplexity": { "version": 1, "promptArea": "#ask-input", "promptIsContentEditable": true, "sendButton": "button[aria-label=\"제출\"]", "responseArea": "div[id^=\"markdown-content\"]", "stopButton": "button:has(use[*|href$=\"pplx-icon-player-stop-filled\"])", "generatingFlag": "" }
```
- sendButton **`button[aria-label="제출"]`** ✅ (한글; 영어=Submit, 아이콘 대안 `button:has(use[href$="pplx-icon-arrow-right"])`)
- responseArea **`div[id^="markdown-content"]`** ✅ (답 본문 ID `markdown-content-0/1/…`, 마지막=최신. 공유/복사/후속질문 버튼은 바깥이라 제외됨. draft `div.prose`도 내부 존재하나 ID 쪽이 정확)
- promptArea **`#ask-input`** ✅ (contenteditable, **Lexical 에디터** → `promptIsContentEditable:true`). ⚠️ Lexical은 textContent 주입이 안 먹을 수 있음 → 구현 시 `execCommand('insertText')`/beforeinput·paste 폴백(221 §5). responseArea는 `div.prose`(probe found 1)도 가능.
- stopButton **`button:has(use[*|href$="pplx-icon-player-stop-filled"])`** ✅ (생성 중 제출버튼이 정지 아이콘으로 토글 — 아이콘 기반이라 언어무관). 구현 시 `:has` 대신 `document.querySelector('use[*|href$="pplx-icon-player-stop-filled"]')?.closest('button')`도 가능. done 판정 = stopButton 소멸. 전송 = sendButton 클릭 또는 Enter.
### ✅ gemini.google.com — 입력·응답·정지 확정 (2026-05-29)
```json
"gemini": { "version": 1, "promptArea": "div.ql-editor", "promptIsContentEditable": true, "sendButton": "", "responseArea": "div.response-content", "stopButton": "button[aria-label=\"대답 생성 중지\"]", "generatingFlag": "" }
```
- promptArea `div.ql-editor`(CE) ✅ / responseArea `div.response-content` ✅ (sample "Gemini의 응답 …" 캡처) / stopButton **`button[aria-label="대답 생성 중지"]`** ✅ (한글)
- done 판정 = stopButton 소멸. generatingFlag(`div.thinking`) 불필요(0).
- ⚠️ stopButton 언어종속: 영어 UI 대비 robust 대안 = `button:has(mat-icon[fonticon="stop"])`(아이콘 기반·언어무관). Angular class(`mat-…`)/`_ngcontent-…`는 불안정 → 금지.
- sendButton 빈값 = **Enter 전송 가정 → 구현 시 확인** 필요. Enter가 줄바꿈이면 입력 후 send 화살표 버튼 selector 확보.
### ✅ chat.deepseek.com — 입력·응답 확정 (2026-05-29) / send=Enter, stop=best-effort
```json
"deepseek": { "version": 1, "promptArea": "textarea", "promptIsContentEditable": false, "sendButton": "", "responseArea": "div.ds-assistant-message-main-content", "stopButton": "", "generatingFlag": "" }
```
- promptArea `textarea`(페이지에 1개, 깔끔) ✅ / responseArea **`div.ds-assistant-message-main-content`** ✅ (assistant 전용 + `ds-` 안정 class. 부모 `div.ds-message`는 user/assistant 공용→비권장)
- 전송 = Enter(가정, 구현 시 확인). done 판정 = **debounce 1500ms**(깔끔한 stopButton 없음).
- ⚠️ send/stop = **`<div role="button">`**(진짜 `<button>` 아님 → `querySelectorAll('button')`로 못 잡힘) + class 공용(`ds-icon-button`)·해시(`_…`)라 selector 어려움. 동일 토글버튼(↑전송↔■정지). STOP은 구현 때 composer-scoped selector로 다듬기(예: 입력영역 내 `div[role="button"].ds-icon-button`). 미구현이어도 debounce로 완료감지 OK.
### ✅ grok.com — 4종 전부 확정 (2026-05-29)
```json
"grok": { "version": 1, "promptArea": "textarea", "promptIsContentEditable": false, "sendButton": "button[data-testid=\"chat-submit\"]", "responseArea": "div.response-content-markdown", "stopButton": "button[aria-label=\"모델 응답 중지\"]", "generatingFlag": "" }
```
- promptArea `textarea`(found 1) ✅ / responseArea **`div.response-content-markdown`** ✅ (답 본문만; "생각함"·버튼 제외). 대안(더 안정): **`[data-testid="assistant-message"]`**(단 "N초 생각함" 텍스트 포함). 응답 래퍼는 `div[id^="response-"]`.
- sendButton **`button[data-testid="chat-submit"]`** ✅ (data-testid·언어무관, `type=submit`, ↑아이콘). Enter로도 전송됨.
- stopButton **`button[aria-label="모델 응답 중지"]`** ✅ (생성 중 표시, 네모 아이콘). done 판정 = stopButton 소멸. ⚠️ 한글 aria-label·언어종속(영어 UI면 "Stop model response" 류). 송신과 달리 stop 상태엔 data-testid 없고 아이콘은 inline path라 aria-label이 최선.
- 비고: Grok은 **reasoning("생각함") 섹션**을 답 앞에 표시. `.response-content-markdown`은 그 뒤 실제 답만 잡음.
