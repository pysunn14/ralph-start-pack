# Codex Ralph Starter Package Guide

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f81d9b30-b292-4b01-8019-4d93715f6c11" />

Codex CLI와 Ralphy를 사용해 제품 기획부터 반복 구현까지 실행하는 최소 가이드입니다.
구현 모델과 추론 강도는 사용자 Codex 설정을 기본으로 상속하며,
선택적으로 태스크별 Draft PR 리뷰·수정·재리뷰 사이클을 사용할 수 있습니다.
- [Guide pptx](https://docs.google.com/presentation/d/1YSHV4lT72-h20uqtvwE0CJfigv7_WoKiI7ePklIlKlo/edit?usp=sharing)
- [Sample Product](https://github.com/minsub0922/ralph-tui-game-sample)


> 이 fork는 Codex CLI 전용입니다. 제품 기획과 Ralphy 실행에 모두 Codex를 사용합니다.

## 0. 필수 도구 확인

```bash
codex --version
codex login status
ralphy --version
```

## 1. 저장소 초기화

프로젝트 저장소 루트에서 실행합니다.

```bash
git init
git config user.name "YOUR_NAME"
git config user.email "YOUR_EMAIL"
```

## 2. Codex로 제품 기획

Codex CLI를 실행합니다.

```bash
codex
```

다음 문서의 전체 내용을 복사해 Codex CLI에 입력합니다.

```text
https://github.com/pysunn14/ralph-start-pack/blob/main/SETUP_PROMPTS.md
```

기획 결과를 검토한 뒤 확정합니다.

```text
확정
```

Codex CLI를 종료합니다.

```text
/exit
```

다음 파일이 생성되었는지 확인합니다.

```text
PRODUCT.md
AGENTS.md
tasks.yaml
ralph.workflow.json
```

생성된 기획 파일을 커밋합니다.

```bash
git add PRODUCT.md AGENTS.md tasks.yaml ralph.workflow.json
git commit -m "docs: define product and Ralph tasks"
```

## 3. Ralphy 초기화
intall: https://github.com/michaelshimeles/ralphy

seed scaffold: 제품 기능이 아닌 프로젝트 뼈대만 먼저 설정

```bash
ralphy --no-commit --max-retries 1 --no-browser \
  --codex \
  "Read PRODUCT.md, tasks.yaml, and AGENTS.md.
   Create only the minimal runnable seed project.
   Do not implement product features.
   Add npm run check and make it pass."
```

scaffold 이후 init

```bash
ralphy --init
```

## 4. Ralphy 규칙 등록

아래 명령 전체를 한 번에 복사해 실행합니다.

```bash
ralphy --add-rule \
  "Before coding, read PRODUCT.md and AGENTS.md and identify the target TUI behavior, keybindings, states, and acceptance criteria"

ralphy --add-rule \
  "Work on exactly one task and avoid unrelated refactoring or changes to unrelated TUI screens"

ralphy --add-rule \
  "Preserve the existing TUI architecture, state management, rendering conventions, and keybinding patterns unless the task explicitly requires changing them"

ralphy --add-rule \
  "Do not add, remove, or update dependencies"

ralphy --add-rule \
  "Handle terminal input, resize events, small terminal sizes, long text, empty states, and Unicode text without crashing or corrupting the layout"

ralphy --add-rule \
  "Restore terminal state on normal exit, cancellation, errors, and SIGINT; do not leave raw mode, cursor visibility, or alternate-screen state corrupted"

ralphy --add-rule \
  "Do not write debug logs or unstructured output over the active TUI; use the repository's existing logging mechanism"

ralphy --add-rule \
  "Keep rendering, keyboard handling, and state transitions deterministic and testable without timing-dependent sleeps or assertions"

ralphy --add-rule \
  "Before reporting task completion, run the repository-defined tests and validation commands relevant to the task, including TUI rendering, input handling, state transitions, and terminal cleanup where applicable"

ralphy --add-rule \
  "Do not weaken, skip, delete, or rewrite tests merely to make them pass"

ralphy --add-rule \
  "Do not modify SETUP_PROMPTS.md, PREFLIGHTS.md, PRODUCT.md, AGENTS.md, ralph.workflow.json, ralph.environment.json, scripts/ralph-preflight.sh, or files under .ralphy/preflight"

ralphy --add-rule \
  "Complete exactly one task per iteration and create exactly one descriptive Git commit containing the task implementation, tests, tasks.yaml update, and .ralphy/progress.txt update before reporting completion"
```

## 5. Ralphy 설정 생성

Codex CLI를 다시 실행합니다.

```bash
codex
```

다음 프롬프트를 입력합니다.

```text
Read PRODUCT.md, AGENTS.md, tasks.yaml, and ralph.workflow.json, then populate .ralphy/config.yaml for this project. Protect the workflow and preflight files from feature tasks. Do not implement product features.
```

설정 내용을 확인한 뒤 Codex CLI를 종료합니다.

```text
/exit
```

설정 파일이 생성되었는지 확인합니다.

```bash
cat .ralphy/config.yaml
```

## 6. Preflight 생성과 검증

Codex CLI를 다시 실행하고 `PREFLIGHTS.md`의 전체 내용을 입력합니다.

```bash
codex
```

생성 작업이 끝나면 다음 파일을 확인합니다.

```text
ralph.environment.json
scripts/ralph-preflight.sh
```

현재 환경을 검증합니다.

```bash
./scripts/ralph-preflight.sh plan tasks.yaml
./scripts/ralph-preflight.sh setup tasks.yaml
./scripts/ralph-preflight.sh verify tasks.yaml
```

리뷰 사이클을 사용하려면 push 가능한 GitHub `origin`, GitHub CLI 로그인과
PR 생성·merge 권한이 필요합니다. Preflight가 이를 읽기 전용으로 확인하고,
충족하지 못하면 feature task를 시작하지 않습니다.

## 7. Ralphy 실행

`ralph.workflow.json`에서 리뷰 사이클을 사용하지 않는 경우:

```bash
./scripts/ralph-preflight.sh run tasks.yaml -- \
  --yaml tasks.yaml \
  --max-iterations 1 \
  --max-retries 1 \
  --no-browser
```

리뷰 사이클을 사용하는 경우:

```bash
./scripts/ralph-preflight.sh cycle tasks.yaml -- \
  --yaml tasks.yaml \
  --max-retries 1 \
  --no-browser
```

`manual` merge에서는 PR을 ready 상태로 만든 뒤 멈추고,
사용자가 merge한 후 같은 명령을 다시 실행합니다.
`automatic` merge에서는 최신 커밋의 리뷰, quality command, 설정된 status check와
GitHub merge 상태가 모두 통과한 경우에만 merge하고 다음 태스크로 진행합니다.
