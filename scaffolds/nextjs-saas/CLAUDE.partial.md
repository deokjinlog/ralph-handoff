## 스캐폴드 계약 (nextjs-saas) — 수정 금지 규칙

이 프로젝트는 검증된 golden 스캐폴드 위에 **도메인 로직만** 얹는다. 배관은 이미 검증됐다 — **다시 짜지 마라.**

- **읽기 전용(locked):** `lib/auth/**` · `lib/payments/**` · `middleware.ts` · `app/(login)/**`
  수정이 필요하다고 판단되면 `BLOCKED.md` 에 사유를 적고 건너뛴다. **우회 구현도 금지.**
- **도메인 로직은 여기만:** `app/(dashboard)/**` · `lib/db/schema.ts` · `components/features/**`
- **새 UI 는 shadcn/ui 기존 컴포넌트 조합으로만.** 커스텀 CSS 파일 생성 금지 — Tailwind 유틸리티 클래스만.
- **의존성 추가는 아래 허용 목록에 있는 것만.** (허용 목록 밖 의존성·`locked_paths` 수정은 `PROMPT.md` 의 6-6 규칙이 `BLOCKED.md` 로 강등한다)
- **dev 서버:** `pnpm dev` — 브라우저 검증(6-8)의 능력 감지가 이 명령을 쓴다.

> 이 계약은 루프가 생성하는 코드의 표면적을 줄인다. **표면적이 줄수록 덜 틀린다.**
