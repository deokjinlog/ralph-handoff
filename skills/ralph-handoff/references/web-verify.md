# 브라우저 검증 — `[e2e]` AC 를 무인 루프가 실제로 클릭하게 하기

> SKILL.md Step 6-8 의 상세본. **핸드오프 세션이 `PROMPT.md` 를 쓸 때** 이 규칙을 그대로 박는다.
> 핵심 성질: **완전 additive.** 브라우저를 못 띄우면 오늘과 동일(BLOCKED). 띄우면 한 단계 위.

---

## 왜 CLI 인가 (MCP 아님)

무인 루프는 **컨텍스트가 생명**입니다. Playwright 를 MCP 서버로 붙이면 매 반복 도구 스키마·서버 상태가 컨텍스트를 먹습니다. `playwright-mcp` README 자체가 코딩 에이전트에는 **CLI + Skill 방식**을 권합니다. 그래서 루프는 `npx playwright` 를 **bash 로** 직접 돕니다 — 붙여둘 서버가 없습니다.

---

## 능력 감지 — `PROMPT.md` 의 6-8 첫 동작

루프가 매 반복 이걸 먼저 판정합니다. **하나라도 아니면 브라우저 검증은 건너뛰고 `[e2e]` AC 는 BLOCKED 로.**

```bash
# 1) Playwright 러너가 있나 (없으면 설치 시도 — 실패해도 루프를 죽이지 마라)
npx playwright --version >/dev/null 2>&1 || npx playwright install --with-deps chromium >/dev/null 2>&1
npx playwright --version >/dev/null 2>&1 || { echo "no-playwright"; }

# 2) dev 서버 기동 명령이 CLAUDE.md 에 있나 (playwright.config 의 webServer 로 쓴다)
grep -iE 'dev 서버|npm run dev|pnpm dev|uvicorn|flask run|next dev' CLAUDE.md >/dev/null 2>&1 || echo "no-devserver"
```

- **둘 다 OK** → 아래 "검증 실행" 으로.
- **하나라도 실패** → `[e2e]` AC 를 `BLOCKED.md` 로. 확인 절차(주소·입력·기대 화면)를 적어서. **전체 루프는 계속.** (오늘과 동일)

> 헤드리스 환경에서 브라우저 바이너리가 안 깔리는 건 흔합니다. **그건 실패가 아니라 fallback** 입니다 — BLOCKED 로 내리고 다음으로.

---

## AC → Playwright 테스트 번역

`[e2e]` 태그가 붙은 AC 하나 = `e2e/` 아래 테스트 하나. **AC 문장을 그대로 시나리오로.**

```
AC-3 [e2e]: 가입 폼에 이메일·비번을 넣고 제출하면 대시보드로 이동한다
```
↓
```ts
// e2e/ac-3-signup.spec.ts
import { test, expect } from '@playwright/test';

test('AC-3: 가입하면 대시보드로 이동한다', async ({ page }) => {
  await page.goto('/signup');
  await page.getByLabel('이메일').fill('t@e.st');
  await page.getByLabel('비밀번호').fill('pw12345!');
  await page.getByRole('button', { name: '가입' }).click();
  await expect(page).toHaveURL(/\/dashboard/);          // ← AC 의 "예/아니오" 가 곧 assertion
});
```

**규칙:**
- 테스트 이름에 **AC 번호를 박아라** (`AC-3: …`). 추적에 쓴다.
- assertion 은 AC 의 "예/아니오" 를 **그대로** 옮긴 것이어야 한다. "페이지가 뜬다" 로 도망가지 마라 — 그건 렌더 증거지 정답 증거가 아니다.
- 셀렉터는 `getByRole`/`getByLabel` 같은 **접근성 기반**을 써라. CSS 클래스로 잡으면 확인용 화면의 사소한 변경에도 깨진다.
- `[unit]` AC 는 여기 오지 않는다 — pytest/vitest 자리다.

---

## 검증 실행 — `PROMPT.md` 에 박을 조각 (그대로 복사)

```markdown
### 브라우저 검증 (VERIFY.md 증거 강제의 확장)

능력 감지가 통과했을 때만 한다. 하나라도 실패하면 `[e2e]` AC 를
BLOCKED.md 로 내리고(확인 절차째) 이 절을 건너뛴다 — 전체 루프는 멈추지 않는다.

- 요구사항의 각 `[e2e]` AC 는 `e2e/` 아래 Playwright 테스트로 존재해야 한다.
  없으면 그 AC 는 미완료다.
- playwright.config 의 `webServer` 로 dev 서버를 자동 기동한다. `retries: 1` 고정.
- 검증 명령:
      npx playwright test --reporter=line
  이 명령의 **터미널 출력 전문**을 VERIFY.md 에 붙인다. 요약·재작성 금지.
- 실패 시 `test-results/` 의 스크린샷·trace 경로도 VERIFY.md 에 남긴다.
  스크린샷은 증거다 — 지우지 마라.
- flaky (retries 후에도 결과가 흔들림) 판정이면, 그 AC 를 무한 재시도하지 말고
  BLOCKED.md 로 강등한다.

**완료 판정:** `[e2e]` AC 는 (테스트 통과) 이거나 (BLOCKED.md 에 기록됨) 이면
완료로 친다. 브라우저를 못 띄우는 것을 완료 조건에 걸어 루프를 매달지 마라.
```

---

## `playwright.config.ts` 최소본 (스캐폴드에 없으면 루프가 만든다)

```ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  retries: 1,                          // flaky 를 한 번은 흡수, 그 이상은 BLOCKED
  use: { baseURL: 'http://localhost:3000', screenshot: 'only-on-failure', trace: 'retain-on-failure' },
  webServer: {                         // 능력 감지에서 찾은 dev 명령
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: true,
    timeout: 120_000,
  },
});
```

---

## 하지 말 것

| 안티 패턴 | 왜 |
|---|---|
| `expect(page).toBeTruthy()` 류 사소한 assertion | 렌더 증거지 정답 증거가 아니다. AC 의 예/아니오를 옮겨라 |
| 브라우저 없는 걸 완료 조건에 걺 | 루프가 영원히 안 끝난다. 통과 OR BLOCKED = 완료 |
| flaky 를 무한 재시도 | 헛돈다. `retries:1` 넘으면 BLOCKED 강등 |
| MCP 서버로 Playwright 물기 | 매 반복 컨텍스트만 먹는다. CLI 로 |
| 스크린샷·trace 삭제 | 아침에 사람이 볼 유일한 렌더 증거다 |
| CSS 클래스 셀렉터 | 확인용 화면 사소한 변경에 깨진다. role/label 로 |

---

## 이건 무엇을 바꾸고, 무엇을 안 바꾸나

- **바꾼다**: `눈으로 확인 ❌` → `pytest 출력만` 이던 조각이, 브라우저가 있으면 **기계가 클릭한 렌더 증거**까지 남긴다.
- **안 바꾼다**: 브라우저가 없으면 정확히 오늘과 같다 — BLOCKED.md + 사람 확인 절차. 골든 스캐폴드·오케스트레이터와 **독립**이다. 스캐폴드가 dev 서버 명령을 `CLAUDE.md` 에 넣어주면 능력 감지가 더 자주 통과할 뿐.
