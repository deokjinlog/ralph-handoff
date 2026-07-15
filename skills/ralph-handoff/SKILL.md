---
name: ralph-handoff
description: Use when the user wants to hand work to an unattended loop — "랄프 돌려줘", "랄프한테 넘겨", "자고 일어나면 돼 있게", "무인으로 만들어줘", "ralph-loop 준비", or explicitly invokes /ralph-handoff. Walks the user through AskUserQuestion gates to produce the three files an unattended loop needs (CLAUDE.md = what to build it with, requirements = what to build, PROMPT.md = how the loop runs), isolates them in a git worktree, and prints the exact ralph-loop command. Never starts the loop itself.
---

# Ralph Handoff — 요구사항까지만 승인하고 나머지를 맡긴다

**당신은 "뭘 만들지"까지만 정합니다. 나머지는 자고 일어나면 돼 있습니다.**

```
당신 │ 기획 자료 (있으면) → 질문 몇 개 → 승인 2번        ← 여기까지
     ▼
랄프 │ 개발방향 → 구현계획서 → 코드 → 검증               ← 무인
```

<HARD-GATE>
이 스킬은 **워크트리를 만들고 파일을 지웁니다.** 그래서 **파괴적 동작 전에 반드시 `AskUserQuestion` 으로 승인**을 받습니다. 승인 없이 `git worktree` · `git rm` · `rm` 을 실행하지 마세요.

**루프를 직접 시작하지 마세요.** 마지막에 명령을 **출력만** 합니다. 루프는 사람이 겁니다 — `ralph-loop` 가 그 명령을 `hide-from-slash-command-tool` 로 막아둔 이유이기도 합니다.
</HARD-GATE>

## 전제 — `ralph-loop` 플러그인이 필요합니다

없으면 아래를 안내하고 중단하세요.

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin install ralph-loop@claude-plugins-official
```

## Checklist

각 항목을 `TaskCreate` 로 만들고 순서대로 완료하세요.

1. **어디까지 맡길지 확인** — Step 1
2. **기획 자료 확인** — Step 2
3. **요구사항 확보** — Step 3 (없으면 `brainstorming` 위임)
4. **`CLAUDE.md` 확보** — Step 4 (★ 가장 자주 빠지는 것)
5. **워크트리 생성** — Step 5
6. **`PROMPT.md` 작성** — Step 6
7. **실행 명령 출력** — Step 7 (직접 실행 X)

---

## ★ 무인 루프가 받아야 하는 건 3개다

**루프는 채팅을 못 봅니다. 파일에 없으면 없는 겁니다.** 이 스킬의 존재 이유가 이 표입니다.

| 파일 | 답하는 질문 | 없으면 |
|---|---|---|
| **`CLAUDE.md`** | **뭘로** 만드나 (스택 · 구조 · 원칙) | **루프가 스택을 지어낸다** |
| **`<slug>-requirements.md`** | **뭘** 만드나 (기술 중립) | 만들 게 없다 |
| **`PROMPT.md`** | 루프가 **어떻게 도나** | 무한 루프 / 거짓 완료 |

> **실제 사고** — `CLAUDE.md` 없이 넘길 뻔했습니다. 요구사항은 **일부러 기술 중립**이라 (`intent-locked-workflow` 의 `brainstorming` 규칙: *"기술 결정은 여기 박지 마세요"*) 스택 키워드가 **0건**이었어요. 스택이 **채팅에만** 있었던 겁니다. 그대로 돌렸으면 Flask 든 Express 든 루프가 자기 맘대로 골랐습니다.
>
> **요구사항이 기술 중립인 건 버그가 아니라 규칙입니다.** 스택은 원래 `CLAUDE.md` 자리예요.

---

## Step 1 — 어디까지 맡길지

`AskUserQuestion`:

```json
{
  "question": "어디까지 무인 루프에 맡길까요?",
  "header": "맡길 범위",
  "multiSelect": false,
  "options": [
    {"label": "요구사항만 주고 나머지 전부 (권장)",
     "description": "설계·계획·코드를 루프가 만듭니다. 기술 선택권을 넘기는 대신 당신은 승인 2번만"},
    {"label": "계획까지 사람이, 코드만 루프에게",
     "description": "설계·계획을 직접 검토하고 승인합니다. 기술 선택이 중요한 프로젝트라면 이쪽"}
  ]
}
```

**"코드만" 을 고르면** — 요구사항·개발방향·구현계획서가 **전부 승인된 상태**여야 합니다. 하나라도 미승인이면 그걸 알리고 중단하세요.

## Step 2 — 기획 자료

`AskUserQuestion`:

```json
{
  "question": "기획 자료가 있나요? 있으면 그걸 읽어서 요구사항과 스택을 뽑습니다.",
  "header": "기획 자료",
  "multiSelect": false,
  "options": [
    {"label": "있습니다 — 파일이 프로젝트 안에 있어요",
     "description": "pptx · pdf · 워드 · md 뭐든. 다음 턴에 경로를 알려주시면 읽습니다"},
    {"label": "있습니다 — 지금 말로 설명할게요",
     "description": "채팅으로 붙여넣거나 설명해주세요"},
    {"label": "없습니다 — 대화하면서 정할게요",
     "description": "질문을 주고받으며 요구사항을 만듭니다"}
  ]
}
```

**파일이 있다면 실제로 읽으세요.** `.pptx` 는 zip 이라 파이썬으로 텍스트를 뽑을 수 있습니다:

```bash
python3 -c "
import zipfile, re, sys, glob
z = zipfile.ZipFile(glob.glob(sys.argv[1])[0])
slides = sorted([n for n in z.namelist() if re.match(r'ppt/slides/slide\d+\.xml$', n)],
                key=lambda x: int(re.search(r'\d+', x.split('/')[-1]).group()))
for i, s in enumerate(slides, 1):
    t = re.sub(r'<[^>]+>', ' ', z.read(s).decode('utf-8', 'ignore'))
    print(f'--- slide {i}\n{re.sub(chr(92)+chr(115)+chr(43), chr(32), t).strip()}')
" '<경로>'
```

**읽은 내용을 요약해서 보여주고 "이게 맞나요?" 확인하세요.** 원본 기획서는 `.gitignore` 로 막는 걸 권합니다 — **기획서는 로컬에, 뽑아낸 요구사항만 git 에.**

## Step 3 — 요구사항 확보

```bash
F=$(ls -d docs/features/*-<slug> 2>/dev/null | head -1)
sed -n '/^## 변경이력/,$p' "$F"/*-requirements.md 2>/dev/null | grep -c '^- \*\*id\*\*:'
```

**변경이력 엔트리가 1건 이상 = 승인 완료.** `intent-locked-workflow` 가 쓰는 신호를 그대로 씁니다 (*"If ANY entry exists, this is NOT initial creation"*).

| 상태 | 할 일 |
|---|---|
| 엔트리 ≥ 1 | ✅ 통과 — Step 4 로 |
| 파일은 있는데 엔트리 0 | **쓰다 만 것.** 승인부터 받으세요 |
| 파일 없음 · `intent-locked-workflow` 설치됨 | **`brainstorming` 스킬에 위임** — Step 2 에서 확인한 기획 자료를 입력으로 |
| 파일 없음 · 미설치 | 사용자와 직접 요구사항을 만드세요 (아래 최소 형식) |

**`brainstorming` 에 위임했다면** — 그게 끝나고 자동으로 `tech-design` 으로 넘어가려 합니다. **Step 1 에서 "요구사항만" 을 골랐으면 거기서 멈추게 하세요.**

### 요구사항 최소 형식 (직접 만들 때)

기술 용어를 **넣지 마세요.** 스택은 `CLAUDE.md` 자리입니다.

```markdown
## 3. 기능 요구사항 (FR)
- **FR-1**: <시스템이 ~한다>          ← 번호 필수. 나중에 추적에 씁니다
## 4. 비기능 요구사항 (NFR)
- **NFR-1**: <얼마나 / 어디까지 — 숫자>
## 5. 범위 밖
- <안 만들 것>                        ← 이게 루프의 울타리입니다
## 6. 수용 기준 (AC)
- **AC-1**: <예/아니오로 답할 수 있게>   ← 그대로 테스트가 됩니다
```

## Step 4 — ★ `CLAUDE.md` 확보 (가장 자주 빠지는 것)

```bash
test -f CLAUDE.md && grep -ciE 'FastAPI|Django|Express|Spring|React|Vue|Next|Postgres|MySQL|Neo4j|Rails' CLAUDE.md
```

**있고 스택이 잡히면** 한 줄 보고하고 넘어갑니다.

**없거나 스택이 0건이면 — 여기서 멈추고 만듭니다.** 근거는 이 순서로:

1. **기획 자료** (Step 2) — 고정된 스택이 적혀 있는 경우가 많습니다
2. **기존 코드** — `package.json` · `pyproject.toml` · `requirements.txt` · import 문
3. **둘 다 없으면 `AskUserQuestion` 으로 물어보세요.** 지어내지 마세요

초안 형식:

```markdown
# <프로젝트> — 프로젝트 컨텍스트

> 이 파일은 "**뭘로** 만드나" 를 담는다.
> "**뭘** 만드나" 는 docs/features/<날짜>-<슬러그>/*-requirements.md 에 있다.
> 요구사항은 일부러 기술 중립이다. 스택을 지어내지 말고 여기를 봐라.

## 무엇을 만드나          ← 한 문단 + 이 프로젝트가 지키는 것 한 줄
## 고정된 스택 — 임의로 바꾸지 마라
| 층 | 무엇 |            ← 프런트 · 백엔드 · DB · 인프라
**명세에 없는 기술을 도입해야 하면 먼저 멈추고 기록해라.**
## 폴더 구조              ← 트리
## 설계 원칙              ← 이 프로젝트가 다른 이유
## 코드 규칙              ← 언어 · 주석 · 테스트 위치
## 지금 단계에서 하지 않는 것   ← 요구사항 §5 와 일치시킬 것
```

**초안을 보여주고 `AskUserQuestion` 으로 승인받으세요.** 스택은 사람이 확정해야 합니다 — 루프가 고르면 되돌릴 수 없어요. 승인 후 커밋합니다.

## Step 5 — 워크트리 생성

**파괴적 동작입니다. `AskUserQuestion` 으로 승인받으세요.** 뭘 만들고 뭘 지울지 목록으로 보여준 뒤에요.

```bash
git status --porcelain          # 비어 있어야 함. 아니면 커밋/stash 안내 후 중단
W="../$(basename $(pwd))-ralph-<slug>"
git worktree add "$W" -b "ralph/<slug>"
```

**"요구사항만" 모드면** — 워크트리 **안에서만** 하류 산출물을 지웁니다. 원본 브랜치는 그대로입니다.

```bash
cd "$W"
git rm -q "docs/features/"*"-<slug>/"*-tech-design.md \
          "docs/features/"*"-<slug>/"*-implementation-plan.md 2>/dev/null || true
# 그 피처가 만든 코드가 있으면 함께 (계획서의 Files 목록을 근거로 판단)
git commit -q -m "chore: 무인 실행용 — 요구사항만 남김"
```

**`-requirements.md` 와 `CLAUDE.md` 는 절대 지우지 마세요.** 그게 넘기는 대상입니다.

이미 워크트리가 있으면 **덮어쓰지 말고** 경로를 보고한 뒤 중단하세요.

## Step 6 — `PROMPT.md` 작성

`$W/PROMPT.md` 를 만듭니다. **아래는 전부 실제 무인 실행에서 터진 것들입니다. 하나도 빼지 마세요.**

### 6-1. 상태 판정 — 매 반복 첫 동작

루프는 매 반복 **기억이 없습니다.** 진척을 파일에서 읽게 하세요.

```
② 없음 / 변경이력 0        → ② 를 쓴다 → 커밋 → 멈춘다
② 완성 · ③ 없음 / 0        → ③ 을 쓴다 → 커밋 → 멈춘다
③ 완성 · REVIEW.md 없음    → ③.5 검문 → 커밋 → 멈춘다
REVIEW.md 있음             → 체크 안 된 첫 task
전부 체크 · VERIFY.md 없음  → 검증 돌리고 VERIFY.md
```

**변경이력 엔트리 수가 완성 신호입니다.** `auto-tech-design` 은 **커밋을 안 해서** git 이력으로는 진척을 못 읽습니다.

### 6-2. 한 반복 = 한 단계

`auto-tech-design` 은 끝나면 `auto-writing-plans` 를 **자동 호출**합니다 (Step 7). **그걸 하지 말라고 명시하세요.** 안 그러면 한 반복에 다 하려다 컨텍스트가 터지고, 반쯤 쓰다 죽으면 다음 반복이 그걸 완성본으로 오해합니다.

### 6-3. ★ 완료 조건에 사람이 센 숫자를 넣지 마라

```
❌  pytest 25건 통과
✅  pytest 전부 통과 (실패 0 · 에러 0)
    + 계획서가 정의한 테스트가 전부 수집됨:
      grep -oE 'def test_[a-z0-9_]+' <계획서> | sort -u | wc -l
        ==  pytest --collect-only -q | grep -c '::test_'
```

**실제로 터졌습니다** — 계획서에 `22건` 이라 적혔는데 본문이 정의한 건 `25건` 이었습니다. **사람이 안 세면 틀릴 수가 없습니다.**

### 6-4. ★ ③.5 검문 관문

**루프가 쓴 계획서로 루프가 채점하면, 오류가 사라지는 게 아니라 안 보이게 됩니다.**

`Task` 도구로 서브에이전트를 **구현자가 아닌 검문자**로 띄우게 하세요:

> 이 계획서만 읽어라. **코드는 절대 보지 마라.**
> 네 목적은 구현이 아니라 **어긋남 찾기**다.
> 1. 완료 조건의 **모든 숫자**를 본문과 대조해라
> 2. 본문에 없는 파일명·함수명이 완료 조건에 있는지 봐라
> 3. 계획서가 정의한 테스트를 **직접 세어** 비교해라
> 4. 요구사항의 FR·NFR·AC 중 **어느 task 에도 안 걸린 게** 있는지 봐라
> 어긋남을 전부 나열해라. 없으면 `어긋남 없음` 한 줄만.

결과를 `REVIEW.md` 에 그대로 적고, 어긋남이 나오면 **계획서(정본)를 고친 뒤** 진행. **코드에서 우회 금지.**

**다른 프롬프트 · 다른 컨텍스트 · 다른 목적** — 이 셋이 달라야 가로질러 읽힙니다.

### 6-5. ★ 나가도 증거가 남게

`ralph-loop` 의 Stop 훅은 상태파일이 사라지면 종료를 못 막습니다. 그런데 훅은 **9군데**에서 그 파일을 지웁니다 — promise 감지·최대반복만 정상이고, **transcript 파싱 실패 4개 + 상태파일 손상 3개는 조용히 죽습니다.** 프롬프트로 못 막아요.

**막으려 하지 말고 증거를 남기게 하세요:**

> 마지막에 완료 조건의 명령을 **하나씩 실제로 돌리고**, `VERIFY.md` 에 **터미널 출력을 그대로** 붙여넣어라. 요약·재작성 금지.

**`VERIFY.md` 가 있으면 진짜 돌린 거고, 없으면 안 돌린 겁니다.** promise 는 말이지만 이건 출력입니다.

### 6-6. 사람이 없다 — 묻지 마라

게이트 워크플로는 **"막히면 멈추고 물어봐"**, 루프는 **"막히면 계속 시도"**. **정반대입니다.**

**답할 사람이 없습니다.** 질문하고 기다리면 → 멈춤 → 루프가 같은 프롬프트 재투입 → 또 질문. **무한 루프입니다.**

> **질문하지 마라. `BLOCKED.md` 에 적고, 건너뛰고, 다음으로 가라.**

| 원래 규칙 | 여기선 |
|---|---|
| 계획에 없는 파일을 만져야 함 | `BLOCKED.md` → 건너뜀 |
| 파괴적 작업 (`rm -rf` · `reset --hard` · force-push) | **절대 금지** |
| 자가 복구 3회 실패 | `BLOCKED.md` → 건너뜀 |
| 외부 호출 (`git push` · PR · 외부 API) | **절대 금지.** 로컬 커밋까지만 |
| 계획에 없는 새 의존성 | `BLOCKED.md` → 건너뜀 |
| **요구사항이 틀린 것 같음** | **`BLOCKED.md` → 멈춤. 절대 고치지 마라** |

### 6-7. 금지 목록에 반드시 넣을 것

| 금지 | 왜 |
|---|---|
| **`-requirements.md` 수정** | 요구사항은 사람 것이다. 고치면 이 일의 의미가 사라진다 |
| **`/ralph-loop:cancel-ralph` 호출 · `.claude/ralph-loop.local.md` 삭제·수정** | 완료 안 됐는데 빠져나가는 문 |
| 참이 아닌 `<promise>` | 루프의 유일한 안전장치를 무력화한다 |
| ③.5 검문 건너뛰기 | 자기 답안지로 자기 채점 |
| `VERIFY.md` 를 실제 출력 없이 씀 | 증거가 아니라 거짓말이 된다 |
| 한 반복에 두 단계 | 컨텍스트가 터진다 |
| 커밋 없이 다음으로 | 다음 반복이 진척을 못 읽는다 |
| 테스트를 통과시키려 테스트를 고침 | 테스트가 곧 수용 기준이다 |

## Step 7 — 실행 명령 출력 (직접 실행 X)

```
✅ 준비됐습니다.

  워크트리   ../<repo>-ralph-<slug>
  브랜치     ralph/<slug>
  넘긴 것    CLAUDE.md (스택) · <slug>-requirements.md (FR N개) · PROMPT.md
  원본       그대로입니다 (현재 브랜치 무변경)

이제 새 세션에서 이 두 줄을 치세요 — 루프는 실행한 세션을 그대로 잡습니다:

  cd ../<repo>-ralph-<slug> && claude

  /ralph-loop:ralph-loop "PROMPT.md 를 읽고 그대로 따라라. 매 반복 그 파일이 지시서다." --completion-promise "<완료문구>" --max-iterations 30

⚠️ 명령 이름이 /ralph-loop 가 아니라 /ralph-loop:ralph-loop 입니다
   (플러그인명:명령명). 한 줄로 치세요 — 줄바꿈하면 인자가 깨집니다.

⚠️ --max-iterations 를 꼭 주세요. 완료 문구는 하나만 매칭되므로
   반복 상한이 주 안전장치입니다.

아침에 볼 것:
  cat VERIFY.md     ← 검증을 진짜 돌렸나 (없으면 안 돌린 것)
  cat REVIEW.md     ← 검문자가 뭘 잡았나
  cat BLOCKED.md    ← 막힌 것 · 사람이 볼 것
  git log --oneline
```

**"요구사항만" 모드였다면** 병합 전에 개발방향을 꼭 읽어보라고 안내하세요. 기술 결정을 루프가 골랐습니다.

```bash
git diff main..ralph/<slug> -- 'docs/features/*/*-tech-design.md'
git worktree remove ../<repo>-ralph-<slug>     # 버릴 때
```

---

## 뭘 잃는지 — 시작 전에 알려주세요

| 잃는 것 | 뜻 |
|---|---|
| **설계의 기술 선택권** | `auto-tech-design` 은 *"alternatives 비교 → **recommendation 자동 선택**"* 입니다. 비교는 하지만 **고르는 것도 자기가** 합니다 |
| **계획을 검토할 사람** | 루프가 쓴 계획을 루프가 실행합니다. **자기가 쓴 걸 의심하지 않아요** |

## 이 방식이 안 맞는 경우

- **기술 선택이 중요** → Step 1 에서 "코드만" 을 고르세요
- **요구사항이 모호** → 루프가 추측으로 메웁니다. 요구사항부터 다시
- **되돌리기 어려운 작업 포함** (DB 마이그레이션 · 배포) → 루프는 파괴적 작업이 금지라 전부 `BLOCKED.md` 로 갑니다. 사람이 하세요

## Anti-Patterns

| 안티 패턴 | 이유 |
|---|---|
| `CLAUDE.md` 없이 넘김 | **루프가 스택을 지어낸다.** 이 스킬의 존재 이유 |
| 승인 없이 워크트리·삭제 | 파괴적 동작. `AskUserQuestion` 먼저 |
| 현재 브랜치에서 산출물 삭제 | 원본이 사라진다. **워크트리 안에서만** |
| 루프를 직접 시작 | 사람이 겁니다. 명령을 **출력만** |
| 완료 조건 숫자를 사람이 셈 | 22 vs 25 회귀 |
| ③.5 검문 생략 | 자기 답안지로 자기 채점 |
| 미승인 요구사항으로 넘김 | 잠기는 게 사람 의도가 아니라 AI 추측이 된다 |

## Related

- [`ralph-loop`](https://github.com/anthropics/claude-plugins-official) — 실제 루프 (Stop 훅). **이 스킬은 준비만 하고 시작은 사람이**
- [`intent-locked-workflow`](https://github.com/deokjinlog/intent-locked-workflow) — 요구사항을 만드는 4단계 게이트 워크플로. 있으면 `brainstorming` 에 위임하고, 없으면 최소 형식으로 직접
