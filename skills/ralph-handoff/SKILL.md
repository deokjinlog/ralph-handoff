---
name: ralph-handoff
description: Use when the user wants to hand work to an unattended loop — "랄프 돌려줘", "랄프한테 넘겨", "자고 일어나면 돼 있게", "무인으로 만들어줘", "ralph-loop 준비", or explicitly invokes /ralph-handoff. Reads whatever planning material exists (pptx/pdf/md) and produces the three files an unattended loop needs — CLAUDE.md (what to build it with), requirements (what to build), PROMPT.md (how the loop runs) — then isolates them in a git worktree and prints the exact ralph-loop command. Announces its question plan up front, then stops the user at most twice: slice selection when the planning doc is large, and one combined approval of requirements + CLAUDE.md. Everything else is automatic. Never starts the loop itself.
---

# Ralph Handoff — 요구사항까지만 승인하고 나머지를 맡긴다

**당신은 "뭘 만들지"까지만 정합니다. 나머지는 자고 일어나면 돼 있습니다.**

```
당신 │ (기획서가 크면) 어느 조각? → 요구사항+CLAUDE.md 승인   ← 딱 두 번
     ▼
랄프 │ 개발방향 → 구현계획서 → 코드 → 검증                   ← 무인
```

<HARD-GATE>
**질문을 두 번 넘게 하지 마세요.** 조각 선택(기획서가 클 때만)과 최종 승인, 그게 끝입니다. 사용자는 "알아서 빨리"를 원해서 이걸 씁니다.

**요구사항과 `CLAUDE.md` 는 반드시 승인받으세요.** 루프가 몇 시간 도는데 방향이 틀리면 전부 버립니다. 단 **둘을 묶어서 한 번에** 물으세요.

**현재 브랜치를 절대 건드리지 마세요.** 삭제는 워크트리 안에서만. 원본이 남아야 되돌릴 수 있습니다.

**루프를 직접 시작하지 마세요.** 마지막에 명령을 **출력만** 합니다. 루프는 사람이 겁니다 — `ralph-loop` 가 그 명령을 `hide-from-slash-command-tool` 로 막아둔 이유이기도 합니다.
</HARD-GATE>

## 전제 — 두 가지

### 1. 여기가 프로젝트 폴더여야 합니다

**이 스킬은 `cwd` 를 봅니다.** 자기 플러그인 폴더가 아니라요.

```bash
git rev-parse --git-dir >/dev/null 2>&1 || echo "git 저장소가 아님"
```

git 저장소가 아니면 — **`git init` 을 대신 하지 말고** 아래를 안내하고 중단하세요. 어디에 뭘 만들지는 사용자가 정할 일입니다.

```bash
mkdir -p <프로젝트>/docs && cd <프로젝트>
git init
mv <기획서>.pptx docs/                       # 있으면
printf 'docs/*.pptx\ndocs/*.pdf\n' > .gitignore
git add .gitignore && git commit -m "chore: 초기"
```

> **기획서는 프로젝트 안에 있어야 합니다.** 사용자가 플러그인 폴더에 기획서를 두는 실수가 실제로 있었습니다 — 설치된 플러그인은 `~/.claude/plugins/cache/` 로 복사되는 거라 소스 폴더의 문서는 아무도 못 읽습니다. **이 스킬은 `cwd` 기준으로만 파일을 찾으세요.**

### 2. `ralph-loop` 플러그인이 필요합니다

없으면 아래를 안내하고 중단하세요.

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin install ralph-loop@claude-plugins-official
```

## ★ Step 0 — 시작하자마자 질문 계획을 예고하세요

**사용자가 제일 싫어하는 건 질문 수가 아니라 "언제 끝나는지 모르는 것"입니다.** 첫 응답에 이 한 덩어리를 먼저 내세요. 그래야 사용자가 대응할 준비를 합니다.

```
ℹ️ 여쭐 건 <N>번입니다. 나머지는 알아서 하겠습니다.

   ① 어느 조각을 넘길지    ← 기획서가 <N>장이라 통째로는 PRD 가 안 됩니다
   ② 요구사항 + CLAUDE.md 승인   ← 루프가 몇 시간 돌아서, 여기가 마지막 방어선

   자동으로 할 것: 기획서 읽기 · 스택 추출 · 요구사항 작성 · 커밋 ·
                  워크트리 · PROMPT.md · 실행 명령 출력
```

**기획서가 작아서 ①이 필요 없으면 "1번입니다" 라고 하세요.** 실제 개수를 세서 말하세요 — 지어내지 말고.

이 예고 없이 질문부터 던지면 **사용자는 "이게 언제 끝나지" 하며 지칩니다.**

## Checklist

각 항목을 `TaskCreate` 로 만들고 순서대로 완료하세요.

**질문은 예고한 만큼만. 나머지는 알아서 하세요.**

0. **질문 계획 예고** — Step 0 (★ 첫 응답에)

1. **기획 자료 읽고 조각 제안** — Step 2 *(★ 질문 1 — 기획서가 작으면 생략)*
2. **요구사항 + `CLAUDE.md` 작성** — Step 3·4 (묻지 말고 그냥 쓰기)
3. **둘 다 한 번에 보여주고 승인** — *(★ 질문 2 — 마지막 방어선)*
4. **워크트리 + `PROMPT.md`** — Step 5·6 (승인 없이 자동)
5. **실행 명령 출력** — Step 7 (직접 실행 X)

## ★★ 사용자를 딱 두 번만 멈춰라 ★★

**이 스킬의 성패는 질문 수입니다.** 사용자는 "알아서 빨리"를 원해서 무인 루프를 씁니다. 물을 게 많으면 그냥 자기가 하고 말죠.

| 멈추는 곳 | 왜 못 빼나 |
|---|---|
| **① 조각 선택** *(기획서가 클 때만)* | 30장짜리 발표자료를 어떻게 자를지는 **사람만** 압니다. 통째로는 PRD 가 안 돼요 |
| **② 요구사항 + `CLAUDE.md` 묶어서 1번** | 루프가 **몇 시간** 돕니다. 방향이 틀리면 **전부 버립니다.** 이게 마지막 방어선 |

**그 외는 전부 알아서 하세요. 묻지 마세요:**

| 묻지 말 것 | 대신 |
|---|---|
| "어디까지 맡길까요?" | **기본 = 요구사항만 주고 나머지 전부.** 사용자가 "코드만" 이라 말했을 때만 다르게 |
| "기획 자료 있어요?" | **`docs/` 를 그냥 찾아보세요.** 있으면 읽고, 없으면 그때 한 줄 물어보세요 |
| "PRD? 소크라테스?" · "카테고리?" · "질문 계획 볼까요?" | **`brainstorming` 에 위임하지 마세요.** 게이트가 5개 더 생깁니다. 아래 형식으로 **직접 쓰세요** |
| "워크트리 만들까요?" | **그냥 만드세요.** 원본 브랜치를 안 건드리고 `git worktree remove` 한 줄로 되돌아갑니다 — **파괴적이지 않습니다** |
| "커밋할까요?" | 그냥 하세요. 워크트리 안입니다 |

> **`brainstorming` 위임은 사용자가 명시로 요청할 때만.** 그 스킬은 라우터·모드·카테고리·질문계획·승인으로 게이트가 5개입니다. 여기선 **직접 쓰는 게 맞습니다** — 어차피 사용자가 마지막 승인에서 다 봅니다.

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

## Step 2 — 기획 자료 (묻지 말고 찾으세요)

```bash
ls docs/*.pptx docs/*.pdf docs/*.md 2>/dev/null
```

**있으면 그냥 읽으세요.** 없을 때만 한 줄 물어보세요 — "기획 자료가 있나요? 경로를 알려주시거나, 만들 걸 말씀해주세요." `.pptx` 는 zip 이라 파이썬으로 텍스트를 뽑을 수 있습니다:

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
| 엔트리 ≥ 1 | ✅ 이미 승인된 것 — Step 4 로 |
| 없거나 엔트리 0 | **직접 쓰세요.** 아래 형식으로. 묻지 말고 |

> **`brainstorming` 에 위임하지 마세요** — 사용자가 명시로 요청하지 않는 한. 그 스킬은 라우터·모드·카테고리·질문계획·승인으로 **게이트가 5개 더** 생깁니다. 여기선 직접 쓰는 게 맞아요. 어차피 사용자가 마지막 승인에서 다 봅니다.

### 요구사항 형식 — 이대로 쓰세요

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

**여기서 따로 묻지 마세요.** 요구사항과 함께 **한 번에** 보여주고 승인받습니다 (바로 아래).

## ★ 승인 — 딱 한 번. 여기가 마지막 방어선

**요구사항과 `CLAUDE.md` 를 한 메시지에** 보여주세요. 루프가 몇 시간 도는데 방향이 틀리면 **전부 버립니다.**

보여줄 때 **핵심만** 추려서 — 전문을 다 붙이지 말고:

```
📄 요구사항 — <슬러그>
   FR   <N>개   (한 줄씩)
   NFR  <N>개
   AC   <N>개
   범위 밖  <N>건   ← 이게 루프의 울타리입니다

📄 CLAUDE.md — 고정할 스택
   <층> : <무엇>     (표로)

파일 경로도 같이 적어주세요. 전문은 열어보시면 됩니다.
```

`AskUserQuestion`:

```json
{
  "question": "이대로 무인 루프에 넘길까요? 루프가 이 두 문서만 보고 몇 시간 만듭니다.",
  "header": "넘기기 전 확인",
  "multiSelect": false,
  "options": [
    {"label": "네 — 워크트리 싸주세요",
     "description": "요구사항·CLAUDE.md 커밋 → 워크트리 → PROMPT.md → 실행 명령 출력"},
    {"label": "고칠 게 있어요",
     "description": "뭘 고칠지 말씀해주시면 반영하고 다시 보여드립니다"}
  ]
}
```

**"고칠 게 있어요" 면** — 고치고 **다시 이 게이트 한 번만.** 매번 묻지 마세요.

승인되면 **묻지 말고 끝까지 가세요** (커밋 → 워크트리 → `PROMPT.md` → 명령 출력).

## Step 5 — 워크트리 생성 (승인 후 자동)

**승인받지 마세요. 그냥 만드세요.** 원본 브랜치를 안 건드리고 `git worktree remove` 한 줄로 되돌아갑니다.

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
| 질문 계획을 예고 안 하고 바로 물음 | **이 스킬의 최대 실패.** 사용자는 질문 수보다 **언제 끝나는지 모르는 것**에 지친다 |
| 예고한 것보다 많이 물음 | 예고가 거짓말이 된다. 예고한 만큼만 |
| `brainstorming` 에 위임 (명시 요청 없이) | 게이트가 5개 더 생긴다. 직접 써라 |
| 워크트리 만들기 전에 승인 요청 | 원본 무변경 + 한 줄로 되돌아감. 파괴적이지 않다 |
| 현재 브랜치에서 산출물 삭제 | 원본이 사라진다. **워크트리 안에서만** |
| 루프를 직접 시작 | 사람이 겁니다. 명령을 **출력만** |
| 완료 조건 숫자를 사람이 셈 | 22 vs 25 회귀 |
| ③.5 검문 생략 | 자기 답안지로 자기 채점 |
| 미승인 요구사항으로 넘김 | 잠기는 게 사람 의도가 아니라 AI 추측이 된다 |

## Related

- [`ralph-loop`](https://github.com/anthropics/claude-plugins-official) — 실제 루프 (Stop 훅). **이 스킬은 준비만 하고 시작은 사람이**
- [`intent-locked-workflow`](https://github.com/deokjinlog/intent-locked-workflow) — 요구사항을 만드는 4단계 게이트 워크플로. 있으면 `brainstorming` 에 위임하고, 없으면 최소 형식으로 직접
