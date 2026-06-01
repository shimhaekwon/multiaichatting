# multichat 버그 FIX + selector 진단 정리 (2차)

> **작성일**: 2026-06-01 21:35
> **대상**: `E:\workspace\multichat` (ChatHub v1.45.7 개조, MV3)
> **수정 주체**: Claude (Opus 4.8)
> **선행**: `180559-claude-런타임수정-마무리` 이후 추가 작업
> **검증**: 매 수정마다 `yarn build` ✓ · `tsc --noEmit` 0
> **상태**: 코드 빌드 완료. Grok selector 1건 미해결(진단 대기).

---

## 1. 이번 세션 추가 수정 (180559 이후)

| ID | 문제 | 파일 | 수정 |
|----|------|------|------|
| **B1** | swap 시 과거 LLM이 context에 혼입 | `context-builder.ts` | `participants.includes(llm)` 필터 |
| **B4** | 1R 빈 context(`claude:""`)로 LLM 이상반응 | `context-builder.ts` | 1R은 context 블록 **생략**(`!prevRound→null`) |
| **B2** | 비밀채팅 봇 stale(다른 셀 🔒 연속) | `SecretChatOverlay.tsx` | `botRef`에 `llm` 추적 → 변경 시 재생성 |
| **B3** | 닫아도 요청 미취소 | `SecretChatOverlay.tsx` | `acRef`로 AbortController 보관 → 전송/닫기 시 abort |
| **B5** | STOPPED 경고배너 누락 | `MultiBotChatPanel.tsx` | STOPPED 시 상단 빨강 배너 |
| **B6** | 종료버튼/진행바/전체선택 부재 | `SettingsBar.tsx`·`Sidebar/index.tsx` | 진행바(●●○○) + 세션종료 + 전체선택 |
| **UX** | DONE 후 새 대화/완전 종료 불가 | `use-round-engine.ts`·`RoundActionButton.tsx`·`SettingsBar.tsx` | `engine.reset()` + DONE `[새 대화]` + 세션종료=완전초기화 |
| **Gemini** | 응답 본문이 셀에 안 올라옴 | `adapter-config.json` | `responseArea` `div.response-content`→**`.model-response-text`**(v2). `.response-content`는 "Gemini의 응답" 라벨 포함이라 조기 done |

> (180559 이전: ERR_002 폴링 · RT-1 done fallback · settled 가드 · MutationObserver(백그라운드 탭) · 타임스탬프 · U-1/U-2/U-3/U-5)

## 2. LLM별 selector 현황 (실측 진단)

각 사이트 콘솔에서 응답 본문 selector 실측 결과:

| LLM | 현재 `responseArea` | 상태 |
|-----|---------------------|------|
| ChatGPT | `div[data-message-author-role='assistant']` | ✅ 작동(`.markdown`에도 본문 확인) |
| Claude | `div.standard-markdown` | ✅ 멀티챗 수신됨(작동 추정) — 콘솔 후보엔 없어 0, `.standard-markdown` 확인 권장 |
| Gemini | **`.model-response-text`** (v2 갱신) | ✅ 실측 본문 확인 — **재로드(↻+탭 F5) 후 반영** |
| DeepSeek | `div.ds-assistant-message-main-content` | ⏳ 미검증 |
| Perplexity | `div[id^='markdown-content']` | ⏳ 미검증 |
| **Grok** | `div.response-content-markdown` | ❌ **노후(실측 0개)** — selector 재확인 필요 |

## 3. 미해결 / 다음 작업
1. **Grok `responseArea` 갱신** — Grok 탭 응답 상태에서 본문 selector 추가 진단(후보 6개 모두 0이었음). 후보: `.standard-markdown`/`.prose`/`[class*="markdown"]`/`.whitespace-pre-wrap` 등
2. **Claude `.standard-markdown` 확인** — 콘솔 1개+본문이면 현재 유지 / 0이면 갱신
3. **DeepSeek · Perplexity 미검증** — 사용할 때 동일 방식으로 확인
4. **전체 재로드 검증** — ↻ + app.html 탭 재오픈 + 각 LLM 탭 F5 (Gemini 반영 위해 필수)
5. **ChatGPT 응답 속도** — 85초 등 느림(사이트 자체). done은 정상

## 4. 운영 노트
- **권장 워크플로우**: 각 LLM에 **로그인된 탭을 미리 열어두고** → 멀티챗 열기. 자동 생성 탭은 미로그인/CS 주입 타이밍으로 첫 라운드 실패(ERR_002/Content script not available) 잦음
- **백그라운드 탭 한계**: app.html을 보는 동안 LLM 탭이 비활성이면 사이트가 스트리밍 DOM을 throttle → 실시간 반영 지연(MutationObserver로 타이머 throttle은 회피하나 사이트 throttle은 한계). done은 결국 수신됨
- **재로드 절차**: 코드/selector 변경 후 → 확장 ↻ + **app.html 탭 재오픈**(↻만으론 옛 app 번들 유지) + **LLM 탭 F5**(content script 갱신)
- `DEBUG`(dom-adapter.ts)는 현재 **false**. 진단 시 true로

## 5. git / 빌드 상태
- 멀티챗 소스(`E:\workspace\multichat`): **origin 미설정**(로컬, 커밋 이력 없음) → push 불가. 버전관리 원하면 GitHub repo 연결 필요
- 모든 수정 `yarn build`·`tsc` 통과 상태

---

## 부록. 협업/수정 문서 (20260601)
①121916 ②122721 ③123624 ④124035 ⑤125845 ⑥165054 ⑦180559(런타임 마무리) **⑧213554(본 문서, 버그fix+selector 2차)**

*Claude(Opus 4.8) 코드 수정·빌드·타입검증. 런타임 테스트는 사용자 진행(1~4종 동작, Gemini/Grok selector 진단).*
