<div align="center">

# ralph-handoff

### 요구사항까지만 승인하고, 나머지는 자고 일어나면 돼 있게

<p>
  <img alt="Version" src="https://img.shields.io/badge/version-0.5.0-7c3aed?style=flat-square&labelColor=0d1117">
  <img alt="Claude Code" src="https://img.shields.io/badge/Claude%20Code-Plugin-a78bfa?style=flat-square&labelColor=0d1117">
  <img alt="Requires" src="https://img.shields.io/badge/requires-ralph--loop-f97316?style=flat-square&labelColor=0d1117">
  <img alt="License" src="https://img.shields.io/badge/license-MIT-22c55e?style=flat-square&labelColor=0d1117">
</p>

</div>

<br/>

> ## 무인 루프는 채팅을 못 봅니다.
>
> **파일에 없으면 없는 겁니다.**
> 이 플러그인은 루프가 필요한 **파일 3개**를 갖춰서 워크트리에 싸주고,
> 명령 한 줄을 출력합니다. **루프는 당신이 겁니다.**

---

## 시작 전에 — 프로젝트를 먼저 만드세요

**스킬은 "이미 프로젝트가 있다" 를 전제로 시작합니다.** 이 한 번만 손으로 하세요.

```bash
mkdir -p ~/projects/내프로젝트/docs && cd ~/projects/내프로젝트
git init                                    # ← 워크트리를 뜨려면 git 이 필요합니다
mv ~/어디/기획서.pptx docs/                  # ← 있으면. 없어도 됩니다
printf 'docs/*.pptx\ndocs/*.pdf\n' > .gitignore
git add .gitignore && git commit -m "chore: 초기"
claude
```

> **★ 기획서는 "만들 프로젝트" 안에 두세요.** 플러그인 폴더가 아니라요.
>
> **스킬은 자기 폴더를 안 봅니다. 당신이 있는 폴더(`cwd`)를 봅니다.** 게다가 설치된 플러그인은 `~/.claude/plugins/cache/` 로 복사되는 거라, 플러그인 소스 폴더에 문서를 둬도 **아무도 못 읽습니다.**
>
> **기획서는 `.gitignore` 로 막는 걸 권합니다** — 회사 자료가 공개 저장소에 섞이면 안 되니까요. 뽑아낸 요구사항만 git 에 남습니다.

---

## 쓰는 법

```
/ralph-handoff
```

또는 그냥 **"랄프한테 넘겨줘"** 라고 하면 발동합니다.

**시작하자마자 질문 계획을 먼저 알려줍니다** — 몇 번 물을 건지, 뭘 물을 건지, 나머지는 뭘 알아서 할 건지. 그래야 대응할 준비가 되니까요.

```
     │ ℹ️ "여쭐 건 2번입니다. 나머지는 알아서."   ← 먼저 예고
     │ …docs/ 뒤져서 기획서 읽음…
     │ 기획서가 크면 → "어느 조각 넘길까요?"        ← 멈춤 ①
     │ …요구사항 + CLAUDE.md 작성…
     │ 📄 둘 다 한 번에 → "이대로 넘길까요?"        ← 멈춤 ②
     ▼
     └─ 커밋 → (지킬 코드 있으면) 워크트리 → PROMPT.md → "이 명령을 치세요"
```

**나머지는 안 묻습니다.**

**워크트리는 지킬 코드가 있을 때만** 만듭니다. 빈 프로젝트면 현재 폴더에서 브랜치만 파요 — 지킬 게 없는데 폴더를 늘리면 헷갈리기만 하니까요.

---

## ★ 루프가 받아야 하는 건 3개입니다

| 파일 | 답하는 질문 | 없으면 |
|---|---|---|
| **`CLAUDE.md`** | **뭘로** 만드나 (스택 · 구조) | **루프가 스택을 지어냅니다** |
| **`<slug>-requirements.md`** | **뭘** 만드나 (기술 중립) | 만들 게 없습니다 |
| **`PROMPT.md`** | 루프가 **어떻게 도나** | 무한 루프 / 거짓 완료 |

**이 플러그인이 존재하는 이유가 첫 줄입니다.**

요구사항은 **일부러 기술 중립**입니다 — 그게 게이트 워크플로의 규칙이에요 (*"기술 결정은 여기 박지 마세요"*). 그래서 요구사항만 넘기면 **스택 키워드가 0건**입니다. 루프는 Flask 든 Express 든 자기 맘대로 골라요.

**스택은 원래 `CLAUDE.md` 자리입니다. 자리가 비어 있는 게 문제죠.** 그래서 이 스킬은 **`CLAUDE.md` 부터 확인**하고, 없으면 기획서를 읽어 초안을 만들어 **당신 승인**을 받습니다.

---

## `PROMPT.md` 에 뭐가 들어가나 — 전부 실제로 터진 것들

| 장치 | 뭘 막나 |
|---|---|
| **묻지 말고 `BLOCKED.md`** | 새벽 3시엔 답할 사람이 없습니다. 질문하면 **무한 루프** |
| **한 반복 = 한 단계** | `auto-*` 가 다음 단계를 자동 호출 → 한 번에 다 하려다 **컨텍스트 터짐** |
| **완료 조건에 사람이 센 숫자 금지** | 계획서엔 `22건`, 실제는 `25건` 이었습니다. **기계끼리 대조**시킵니다 |
| **검문 관문** | **자기 답안지로 자기 채점.** 다른 목적의 서브에이전트가 계획서만 가로질러 읽습니다 |
| **`VERIFY.md` 증거 강제** | 훅이 **9군데**에서 루프를 죽입니다 — 막지 말고 **터미널 출력을 남기게** |
| **취소 금지** | `/ralph-loop:cancel-ralph` · 상태파일 삭제는 완료 안 됐는데 빠져나가는 문 |

### 왜 번역이 필요한가

게이트 워크플로는 **"막히면 멈추고 물어봐"**, 루프는 **"막히면 계속 시도"**. **정반대**입니다.

질문하고 기다리면 → 멈춤 → 루프가 같은 프롬프트 재투입 → 또 질문. **무한 루프인데 아무것도 안 만들어집니다.** 그래서 모든 게이트를 *"`BLOCKED.md` 에 적고 건너뛴다"* 로 번역합니다.

---

## 뭘 잃는지 — 먼저 아세요

| 잃는 것 | 뜻 |
|---|---|
| **설계의 기술 선택권** | 루프는 *"대안 비교 → **추천 자동 선택**"* 입니다. 비교는 하지만 **고르는 것도 자기가** 합니다 |
| **계획을 검토할 사람** | 루프가 쓴 계획을 루프가 실행합니다. **자기가 쓴 걸 의심하지 않아요** |

두 번째가 진짜 위험이라 **검문 관문**을 넣었지만 — 완벽하진 않습니다. **아침에 개발방향을 꼭 읽어보세요.**

```bash
git diff main..ralph/<slug> -- 'docs/features/*/*-tech-design.md'
```

### 안 맞는 경우

- **기술 선택이 중요** → "코드만 맡기기" 를 고르세요
- **요구사항이 모호** → 루프가 추측으로 메웁니다
- **되돌리기 어려운 작업** (DB 마이그레이션 · 배포) → 전부 `BLOCKED.md` 로 갑니다. 사람이 하세요

---

## 설치

**`ralph-loop` 가 먼저 필요합니다** — 실제 루프는 그쪽이 돌립니다.

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin install ralph-loop@claude-plugins-official

/plugin marketplace add deokjinlog/ralph-handoff
/plugin install ralph-handoff@ralph-handoff
```

세션을 한 번 재시작한 뒤 `/ralph-handoff`.

### 같이 쓰면 좋은 것

[**intent-locked-workflow**](https://github.com/deokjinlog/intent-locked-workflow) — 기획서를 요구사항으로 만드는 4단계 게이트 워크플로. 있으면 이 스킬이 `brainstorming` 에 위임하고, 없으면 최소 형식으로 직접 만듭니다. **필수는 아닙니다.**

---

## 이 플러그인이 안 하는 것

- **루프를 시작하지 않습니다.** 명령을 출력만 해요 — 사람이 겁니다
- **현재 브랜치를 안 건드립니다.** 전부 워크트리 안에서
- **승인 없이 아무것도 안 지웁니다**

---

<div align="center">
<sub>MIT · <a href="https://github.com/anthropics/claude-plugins-official">ralph-loop</a> 필요</sub>
</div>
