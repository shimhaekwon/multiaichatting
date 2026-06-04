# 227 — 설계 변경안: Floor "속도 경쟁" → "랜덤 순서 직렬" + 발언 순서 표시

> **문서 버전: v1.1** (2026-06-05) · **상태: 구현 완료(`8ce01f5`) · 정본([221]§R2·[222]) 동기화 완료(2026-06-05) · D-D(자동 셔플) 구현 완료**
> **확정 결정**: D-A=**B만(순번 배지)** · D-B=**코드만 먼저**(경량화 후속) · D-C=MD 미포함 · D-D·D-E=후속.
> 대상 코드: `E:\workspace\ai-sarangbang` · 영향 정본: **[221] §R2** · **[222]** · **[223]** · `core-purity.test`
> 변경 이력: 본 문서 **§9** + 폴더 `CHANGELOG.md`. 트리거: 라이브 디버깅 — `EXAONE`·`Qwen3` 연속 "응답 없음".

---

## 0. 한 줄 요약

전원 동시 생성 후 첫 토큰 선점 경쟁(**race**)을, **사람 입력 시 발언 순서를 랜덤 추첨해 한 명씩 직렬 발언**하는 방식으로 바꾸고, 그 **추첨 순서를 Roster 순번 배지로 표시**한다.

---

## 1. 배경 / 문제 — 왜 바꾸나

### 1.1 현행 ([221] §R2 · [222] `raceRound`)
사람 입력 → 후보 AI **전원 동시** `driver.speak()` → **첫 토큰 낸 자가 floor 선점·발언** → 나머지 `abort` → 승자 발언 후 남은 후보 **재경쟁**. 큐 소진까지.

### 1.2 라이브 증상
대화가 길어지면 `EXAONE`·`Qwen3`가 **연속 "응답 없음"**, `Phi4-mini`만 계속 발언.

### 1.3 근본 원인 (측정 근거)
| 지표 | 값 | 해석 |
|---|---|---|
| PROCESSOR | **100% CPU** (전 모델) | GPU 미사용 → 7~8B 추론 느림 |
| RAM | total 31.4GB / **free 7.6GB** | 3모델 워킹셋 **14GB**(5.1+3.0+5.9) > free → 동시 상주 불가 |
| `ollama ps` | `qwen3 ... Stopping...` | 메모리 압박으로 언로드 중 |

**단독 추론 속도** (`/api/generate`, `num_predict=12`): phi4 39.4 / exaone 21.3 / qwen3 19.2 tok·s⁻¹(prompt-eval), 총 3~4s.

**결론**: 단독이면 셋 다 4초 내 정상 → **느림 자체는 직접 원인이 아님**. 진짜 원인은 **동시 race가 저사양 자원(CPU·RAM)을 분할·경쟁**시켜 ① 가장 작은 `phi4` 독식(망가진 결정성), ② 큰 모델은 메모리 경쟁에서 굶어 **60초 idle 타임아웃** → "응답 없음". 즉 **"물리적 동시 추론"이 단일 CPU·저RAM에서 자멸**한다.

### 1.4 설계 철학 재정의
"진정한 대화감" = **① 순서 비결정성 ② 반응 개성 ③ 끼어듦 ④ 맥락 반응**. **물리적 동시 추론은 이 중 어디에도 없으며** ①②의 한 구현일 뿐 → 동시 추론을 포기해도 라이브함은 보존 가능.

---

## 2. 변경 1 — Floor: 랜덤 순서 직렬화

### 2.1 동작
1. 사람 입력 → 발언 후보 확정(`collectSpeakers`, D2 `willSpeak`).
2. **Fisher-Yates 셔플(주입 `rng`)로 순서 추첨.** 정확 형태(테스트 핀, L2):
   ```ts
   private shuffle<T>(arr: T[]): T[] {
     const a = [...arr]
     for (let i = a.length - 1; i > 0; i--) {
       const j = Math.floor(this.rng() * (i + 1)) // rng: () => number ∈ [0,1)
       ;[a[i], a[j]] = [a[j], a[i]]
     }
     return a
   }
   ```
3. 추첨 순서를 `onOrder`로 emit(§3.2) → 전원 `setState('queued')`.
4. 추첨 순서대로 **1명씩 직렬** 발언 — **`speakOne()` 헬퍼**(H1: `streamWinner`의 floor 연출 + `streamMessage` 재사용):
   ```ts
   private async speakOne(id: ParticipantId, signal: AbortSignal): Promise<void> {
     this.room.floorHolder = id          // streamWinner가 하던 floor 연출(streamMessage엔 없음)
     this.setState(id, 'speaking')
     this.notifyRoom()
     await this.streamMessage(id, signal) // 빈응답→status:'error'("응답 없음"), abort→'stopped' 내장
   }
   ```
   > **H1 주의**: `streamMessage`(현 `runAutoTurn` 전용)는 `floorHolder`/`speaking`/`notifyRoom`을 **호출자가** 한다. 직렬 루프에서 그냥 부르면 **현재 화자 하이라이트 누락**(회귀). 반드시 `speakOne`로 감싼다.
5. 루프 ([222] 단일비행 `runTurn` 내):
   ```ts
   const order = this.shuffle(await this.collectSpeakers())
   this.hooks.onOrder?.(new Map(order.map((p, i) => [p.id, i + 1]))) // 1-based rank
   for (const p of order) this.setState(p.id, 'queued')
   for (const p of order) {
     if (this.pending) break            // D3 하드 인터럽트(루프 top, 다음 화자 생성 전)
     const ac = new AbortController(); this.current = ac
     await this.speakOne(p.id, ac.signal)
     this.current = null
     this.spokeThisTurn.add(p.id)
   }
   for (const p of order) if (!this.spokeThisTurn.has(p.id)) this.setState(p.id, 'idle') // M2: 미발언 queued 해제
   this.room.floorHolder = null
   this.hooks.onOrder?.(new Map())        // C2: 배지 클리어
   if (!this.pending) this.room.status = 'idle'
   this.notifyRoom()
   ```

### 2.2 효과
- **자원 독점 → 단독 4초 → "응답 없음" 소멸.**
- **매 턴 셔플 → 순서 비결정 → `phi` 독식 해소.**
- **직렬 보너스(핵심)**: 뒤 순서 발언자가 앞 발언을 `history`로 **본다**(`publishNew`가 즉시 push) → 진짜 대화 연쇄. race(동시 생성)는 서로 못 봄 = 독백 N개.

### 2.3 의도적으로 빠지는 것 (후속 백로그)
- **반응 속도 개성**(순수 random) → 후속 **eagerness 가중 셔플**(구조 변경 없음).
- **끼어듦(바지인)**(순서 확정 후 고정) → 후속. **단 사람 하드 인터럽트(D3)는 유지.**

### 2.4 보존 불변식
- **core 순수성([224]§1)**: 생성자 `opts.rng?: () => number`(기본 `Math.random`) 주입 → 시드 rng로 결정성 테스트.
  > **H3**: 현 `core-purity.test.ts`는 react·document/window만 검사하고 **`Math.random`은 안 막는다**. → **`core-purity.test.ts`에 `/\bMath\.random\b/` 금지 정규식 추가**(§5). 생성자 기본값의 `Math.random` *참조*는 호출이 아니라 무해하나, core에서의 직접 *호출*을 가드.
- **D3 인터럽트(M1)**: `break`는 루프 top(다음 화자 `current` 생성 전). 진행 중 화자는 `startTurn`의 `this.current?.abort()`가 중단 → `streamMessage` catch가 `stopped`. 종료 블록(`floorHolder=null`·`onOrder(new Map())`·status)은 **무조건 실행**.
- **single-flight(`busy`)·whisper·자동대화(C3)**: 그대로(자동 모드 셔플=D-D, **2026-06-05 구현**: `pickAutoSpeaker` 바퀴 로테이션 — 전원 1회씩 균등). 단 자동 턴 진입 시에도 `onOrder(new Map())`로 직전 사람턴 배지 잔류 방지.

---

## 3. 변경 2 — 발언 순서 표시 (D-A = B만)

### 3.1 채택
**Roster 순번 배지** — 각 AI에 추첨 순번 `1·2·3`. 현재 화자는 기존 `floorHolder` 하이라이트로 진행 추적. **방식 A(시스템 라인 발표)는 후속 보류**(§4·§7에서 A 항목 전부 제외).

### 3.2 배선 (C1 — 신규 채널, 5곳)
> 현 `CoordinatorHooks`엔 순번 sink가 **없다**. `turnState`(onState→RoomView→Roster) 패턴을 본떠 신설:

| # | 파일 | 변경 |
|---|---|---|
| 1 | `core/coordinator.ts` `CoordinatorHooks` | `onOrder?(ranks: ReadonlyMap<ParticipantId, number>): void` 추가(**optional** — `onRoom?`처럼 `?.`로 가드) |
| 2 | `app/store.ts` `RoomView` | `turnOrder: ReadonlyMap<ParticipantId, number>` 필드 + 초기 스냅샷 빈 `Map` |
| 3 | `app/store.ts` `createRoomStore` | `onOrder` 구현(`turnOrder` 교체 후 `emit()`) + 반환 객체에 포함 |
| 4 | `ui/Roster.tsx` `RosterProps`/렌더 | `view.turnOrder.get(p.id)` → `<span class="rank">` 배지(seat 옆) |
| 5 | `ui/theme.css` | `.rank` 스타일(90s 토큰) |

> `main.tsx:47`은 `store`를 hooks로 그대로 주입 → `onOrder`가 optional이라 미구현이어도 컴파일은 되지만 **런타임에 순번이 조용히 누락**되므로, store에 **반드시 구현**한다.

### 3.3 생명주기 (C2 — 배지 staleness 차단)
`turnState`와 달리 `turnOrder` Map은 자연 소멸이 없다 → **명시적으로 클리어**:
- **턴 시작**: 새 순번 emit(§2.1 step3).
- **턴 종료**(정상·인터럽트 공통): `onOrder(new Map())` → 배지 제거(§2.1 루프 말미).
- **자동 턴 진입**: `onOrder(new Map())` → 직전 사람턴 배지 잔류 방지.
- 미클리어 시 ①②③가 다음 턴·자동모드까지 **틀린 채 잔류**(이 항목이 빠지면 기능이 눈에 띄게 깨짐).

### 3.4 UX 트레이드오프 / 표기(L1)
순서 선공개 = "누가 먼저?" **서스펜스↓** ↔ 제비뽑기 발표의 **재미↑**(사랑방 사회자 감성). 배지는 **평문 숫자**(`1·2·3`) 권장 — `①②③` 글리프는 `ui-no-korean` 게이트(가-힣만 금지) 통과하나 rank>3 폴백이 없어 숫자가 안전.

---

## 4. 영향 범위 (파일별 — 이번 범위)

| 파일 | 변경 |
|---|---|
| `core/coordinator.ts` | **삭제**: `raceRound`·`pullFirstToken`·`streamWinner`·`indicateNoResponse`(전부 coordinator 내부 전용, 외부 참조 0 — 삭제 안전). **추가**: `shuffle()`(rng)·`speakOne()`(H1)·생성자 `rng` 옵션·`onOrder` emit·`runTurn` 직렬 루프·미발언 queued 해제(M2). |
| `core/coordinator.ts` `CoordinatorHooks` | `onOrder?` 추가(C1 #1) |
| `app/store.ts` | `RoomView.turnOrder` + `onOrder` 구현(C1 #2·#3) |
| `ui/Roster.tsx` | 순번 배지(C1 #4) |
| `ui/theme.css` | `.rank` 스타일(C1 #5) |
| `core/core-purity.test.ts` | `Math.random` 금지 정규식 추가(H3) |
| `core/coordinator.test.ts` | **재작성**(L3, ~5/15): `속도순 선점`·`재생성 calls 1/2/3`(삭제)·`maxStreaming≤1`(불변식 유지·setup 교체)·`(응답 없음) 표식`(**직렬형으로 보존**, H2)·`바지인 D3`(reframe)·describe 제목. **신규**: 시드 rng 순열 검증·미발언 queued 해제·`onOrder` emit/clear. 자동모드 테스트 회귀 확인. |
| **후속(out-of-scope)** | 방식 A(`onSystem`·`Room` 머지 렌더·i18n `sys.turnOrder`·`SystemLine`) · 모델 경량화(D-B) · 자동모드 셔플(D-D) · 반응성/끼어듦(D-E) |

---

## 5. 정본 / 메모리 정합 (✅ 완료 2026-06-05)
- ✅ **[221] §R2/§R3/§4 D4/§5 용어집** "속도 경쟁·FIFO 큐" → **"랜덤 순서 직렬"** 개정(v2).
- ✅ **[222] §1/§2/§4 의사코드/§5/§9/§10** race·queue·enqueue·collectIntents·drainFloor 흐름 → 직렬 흐름 + `shuffle`/`collectSpeakers`/`speakOne`/`onOrder`/자동(`runAutoTurn`·`pickAutoSpeaker`)/`dispose` 갱신(v4).
- ✅ **`core-purity.test.ts`**: `Math.random` 가드 추가(H3, `8ce01f5`).
- ✅ 처리내역 + `MEMORY.md` + 폴더 `CHANGELOG.md` 반영.
- ✅ **[223] `SystemLine`·`Intent` 타입 삭제 완료**(`2af7323`) — 라운드 종료선은 SystemLine 부활이 아니라 **UI 전용 `computeRoundMarks`**로 구현(SystemLine 불요).

---

## 6. 결정 항목 (확정 2026-06-04)

| ID | 항목 | 확정 |
|---|---|---|
| **D-A** | 순서 표시 방식 | **B만** — Roster 순번 배지 (A=시스템 라인은 후속) |
| **D-B** | 모델 경량화 동반? | **코드만 먼저** — 경량화는 효과 본 뒤 별도(R3) |
| **D-C** | MD 저장본에 순서 기록? | **미포함**(불변식 유지) |
| **D-D** | 자동대화(C3) 셔플? | **[구현 2026-06-05]** 바퀴 로테이션(`pickAutoSpeaker` — 전원 1회씩·순서 랜덤·라운드 균등) · 라운드 종료선(`computeRoundMarks`, 전원 한 바퀴) |
| **D-E** | 반응성 가중·끼어듦 | **후속 백로그** |

---

## 7. 구현 순서 (확정)
1. `coordinator`: race 4종 삭제 → `shuffle`·`speakOne`·`rng`·`onOrder`·직렬 루프 + 미발언 queued 해제(M2).
2. `CoordinatorHooks.onOrder?` + `store`(`RoomView.turnOrder`·`onOrder`) + `Roster` 배지 + `.rank` CSS (C1).
3. `core-purity.test` `Math.random` 가드(H3).
4. `coordinator.test` 재작성: 시드 rng 순열·직렬 무응답 보존(H2)·미발언 queued 해제·`onOrder` clear. `tsc` 0·전 테스트 green(자동모드 회귀 확인).
5. **라이브 재현 검증**(Ollama) — "응답 없음" 소멸·순번 배지 표시·턴 종료 시 배지 클리어 확인.
6. 정본([221]§R2·[222])·`core-purity`·처리내역·`MEMORY`·`CHANGELOG` 갱신.

---

## 8. 리스크 / 미해결
- **R1**: 직렬이라 전체 완료까지 **체감 지연↑**(N명 × 단독시간). "응답 없음"보다 나음·타이핑 효과로 완화.
- **R2**: 순수 random이 가끔 부자연 → D-E(eagerness) 후속.
- **R3**: D-B 미동반 시 큰 모델 단독 CPU 추론도 수 초+ → 체감 느림 잔존(굶음은 없음). 경량화로 즉시 해소.

---

## 9. 검토 / 변경 이력 (추적성)

### 9.1 리비전
| rev | 날짜 | 변경 |
|---|---|---|
| draft 0.1 | 2026-06-04 | 초안: race → 랜덤 직렬 + 순서 표시 |
| draft 0.2 | 2026-06-04 | 결정 반영: D-A=B만 · D-B=코드만 먼저 |
| **v1.0** | 2026-06-04 | **최종 검토(uw-critic + 코드 정합 교차) 반영** — 아래 9.2 |

### 9.2 최종 검토 발견 반영 (구현 직전 게이트)
| 발견 | 등급 | 내용 | 반영 위치 |
|---|---|---|---|
| **C1** | CRITICAL | 순번 전달 채널(`onOrder`) 부재 — "store에 전달"이 실제 seam 없음 | §3.2(배선 5곳) |
| **C2** | CRITICAL | 배지 생명주기 미정의 → ①②③ 영구 잔류 | §3.3 |
| **H1** | HIGH | `streamMessage` ≠ `streamWinner`(floor 연출 누락) | §2.1 step4 `speakOne` |
| **H2** | HIGH | "응답 없음" 테스트가 race 모양 강결합 | §4·§7(직렬형 보존) |
| **H3** | HIGH | `core-purity`가 `Math.random` 미금지(phantom 가드) | §2.4·§5(정규식 추가) |
| **M1** | MED | break 시 floor/current 정리 | §2.4 |
| **M2** | MED | 미발언 `queued` 잔류 | §2.1 루프·§4 |
| **M3** | MED | §4가 제외된 A항목 나열 | §4(out-of-scope 분리) |
| **M4** | MED | §5 turn-start "실사용" 거짓 | §5(삭제) |
| **L1~3** | LOW | 배지 표기·셔플 형태 핀·테스트 범위 | §3.4·§2.1·§4 |

> 검토 방식: 독립 `uw-critic` 적대적 검토 + 작성자 직접 코드 정합 검증(`store.ts`·`coordinator.test.ts` 실독) 교차. 판정 FIX-FIRST → 본 v1.0에서 전량 반영.
