## 기술 명세
1. **크롬 확장**
   - Manifest V3 기준
   - 탐지 로직: `chrome.cookies` API로 로그인 상태 확인
   - UI 템플릿: ChatHub 참조

2. **라운드 제어**
```javascript
class RoundController {
  constructor(maxRounds=3) {
    this.current = 0;
    this.max = Math.min(maxRounds, 9);
    this.llms = new Set();
  }
  // 라운드 증감 메서드 구현...
}
```

3. **응답 동기화**
   - Promise.allSettled()로 멀티 LLM 응답 처리
   - 타임아웃: `setTimeout(() => {}, userConfig.timeout)`

4. **저장 시스템**
   - IndexedDB 스키마: 
     ```json
     {
       "session": "YYYYMMDDHHmmss",
       "rounds": [],
       "llms": ["claude","chatgpt"]
     }
     ```
