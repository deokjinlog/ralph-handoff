# 골든 스캐폴드 — 배관을 루프가 안 짜게 (opt-in)

> SKILL.md Step 4·5 의 상세본이자 **절차의 정본**. **웹 SaaS 그린필드일 때만.** 완전 opt-in — 안 쓰면 오늘과 동일.
>
> **★ 이 파일의 모든 `scaffolds/` 경로는 `${CLAUDE_PLUGIN_ROOT}` 기준입니다.** 설치된 플러그인은 `~/.claude/plugins/cache/` 로 복사되므로, `cwd` 아래에서 찾으면 **없습니다** (스킬이 `cwd` 만 본다는 규칙의 예외 — 스캐폴드 자산은 플러그인 안에 있다).

## 왜

엔터프라이즈 SaaS 의 공통 배관(인증·OAuth·멀티테넌시·RBAC·과금·이메일·관리자)은 루프가 매번 새로 짜면 **매번 다르게 틀린다.** 검증된 스타터를 결정론적으로 깔아두면 그 표면적이 0 이 된다 — 루프는 도메인 로직만 얹는다. **"루프 입장의 그린필드"** 를 인위적으로 만드는 장치다.

## 언제 적용하나

- 프로젝트가 **웹 SaaS**(대시보드·인증·구독 과금)이고 **그린필드**(코드 0)일 때.
- `scaffolds/` 에 맞는 스캐폴드가 있을 때 (지금은 `nextjs-saas` 하나).
- **아니면 적용하지 마라.** Python 백엔드·모바일·CLI 등은 스캐폴드 없이 오늘 흐름 그대로. **한 스택은 한 시장만** 커버한다 — 안 맞으면 억지로 끼우지 마라.

## 절차

**1. 채택은 승인 게이트에서 (Step 4)** — 새 멈춤을 만들지 마라. `CLAUDE.md` 초안에 "스캐폴드: nextjs-saas 적용" 을 넣고, 멈춤② 에서 사람이 빼게 한다. (확인용 화면 FR 과 같은 "넣어두고 빼게" 규칙)

> **`scaffold.json` 의 `pin` 이 아직 placeholder 면 자동 채택하지 마라.** 어느 시점 코드가 들어올지 모르는 채로 외부 레포 수백 파일을 얹는 셈이다 — `BLOCKED.md` 에 "스캐폴드 pin 미지정" 으로 남기고 스캐폴드 없이 진행한다.

**2. 클론 + green 베이스라인 (승인 후 Step 5, 격리 안에서):**

```bash
# 격리 폴더엔 이미 CLAUDE.md·requirements·.git 이 있다 → 빈 폴더 클론 불가. 임시로 받아 내용만 얹는다:
git clone --depth 1 <source.url> .scaffold-tmp
rm -rf .scaffold-tmp/.git && cp -rn .scaffold-tmp/. . && rm -rf .scaffold-tmp   # -n: 기존 .gitignore·CLAUDE.md·requirements 보존
pnpm install && pnpm exec tsc --noEmit           # = scaffold.json.verify_bootstrap (자율 green: install + 타입체크)
git add -A && git commit -q -m "chore: 스캐폴드 baseline (green)"
```

**green 이 안 나오면 클론을 버리고 스캐폴드 없이 진행한다** — 사람에게 한 줄 알림. (스타터가 낡았을 수 있다. 깨진 베이스라인 위에 도메인 로직을 얹으면 원인을 못 찾는다.)

> **★ 실측(2026-07-21, `nextjs/saas-starter`):** `pnpm install`·`tsc --noEmit` 은 **자율로 green**. 하지만 `pnpm build`·`pnpm dev` 는 **Postgres·Stripe·AUTH_SECRET env 를 요구**해 죽는다 (`POSTGRES_URL is not set`). `test` 스크립트는 없다. 그래서 **full build/dev green 은 자율로 못 낸다** — `scaffold.json.needs_human_env` 의 DB·결제 셋업은 `BLOCKED.md` 로 사람에게 넘긴다 (이 레포의 *"DB·결제 = 사람 몫"* 경계 그대로). **스캐폴드의 자율 green 은 install + 타입체크까지다.** dev 서버도 env 가 있어야 뜨므로, env 없으면 [e2e] 브라우저 검증(6-8)도 BLOCKED 로 떨어진다.

**3. `CLAUDE.partial.md` 병합** — `${CLAUDE_PLUGIN_ROOT}/scaffolds/nextjs-saas/CLAUDE.partial.md` 를 프로젝트 `CLAUDE.md` 뒤에 붙인다. 이게 `locked_paths` 규칙을 `CLAUDE.md` 에 박는다. **이 병합이 빠지면 락·shadcn 고정·dev 서버 명시가 통째로 사라진다.**

## 락 강제 — 런타임 규칙(6-6), 새 게이트 없음

`locked_paths` 수정 시도·허용 목록 밖 의존성은 **루프의 6-6 런타임 규칙**이 `BLOCKED.md` 로 강등한다 — 사람이 없으니 막지 말고 적고 건너뛴다:

- `locked_paths` 파일을 수정해야 할 것 같음 → `BLOCKED.md` (우회 구현도 금지)
- 허용 목록 밖 의존성 → `BLOCKED.md`

> ③.5 검문 관문은 **코드를 안 본다**(계획서만 읽음). 그래서 코드-레벨 위반(파일 수정·import)은 검문 관문이 아니라 **6-6 런타임 규칙**의 몫이다. 스캐폴드 계약은 `CLAUDE.md` 의 문장 + 6-6 규칙일 뿐 **새 게이트를 안 만든다.**

## 증분1·2 와의 연결

- 스캐폴드가 `dev_server`(`pnpm dev`)를 `CLAUDE.md` 에 넣어줌 → **증분1 브라우저 검증(6-8)의 능력 감지가 더 자주 통과.**
- `shadcn/ui` 고정 → `SKILL.md` 가 지적한 *"루프가 CSS 에 몇 시간"* 을 구조적으로 차단.

## 하지 말 것

| 안티 | 왜 |
|---|---|
| 스타터 코드를 ralph-handoff 에 통째로 vendor | 수백 파일이 썩는다. `scaffold.json` 으로 외부를 pin |
| pin 없이 스타터 최신을 클론 | 어느 날 깨진 걸 클론한다. 커밋 SHA 로 고정 |
| `verify_bootstrap` green 확인 없이 루프 시작 | 깨진 베이스라인 위 도메인 로직 → 원인 못 찾는다 |
| 웹 SaaS 아닌데 스캐폴드 강요 | 한 스택은 한 시장만. 안 맞으면 스캐폴드 없이 |
| `locked_paths` 를 루프가 수정 | 배관이 다시 틀린다. `BLOCKED.md` 로 |
| 스캐폴드 채택에 새 멈춤 추가 | 승인 게이트에서 한 줄로. **멈춤은 여전히 최대 2번** |
