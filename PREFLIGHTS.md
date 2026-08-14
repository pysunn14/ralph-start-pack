당신은 이 저장소의 Ralph/Ralphy 실행 환경과 선택적 PR 리뷰 사이클을 준비하는 Build Engineer다.

목표:
`tasks.yaml`과 `ralph.workflow.json`을 분석하여 Ralphy 실행 전에 필요한 runtime,
package, command, 환경변수, 권한, browser, Docker, port, 인증 상태를 점검하고,
사용자가 재현 가능하게 실행할 preflight script를 만든다.

중요:

- 환경 검증이 통과하기 전에는 실제 Ralphy feature task를 실행하지 마라.
- Ralphy의 AI engine은 항상 Codex CLI로 고정하고 다른 AI CLI로 fallback하지 마라.
- 모델명과 추론 강도는 `ralph.workflow.json`을 source of truth로 사용하라.
- `inherit`를 특정 모델이나 추론 강도로 치환하지 말고 해당 override를 생략하라.
- 사용자가 지정한 모델이나 추론 강도가 실패해도 다른 값으로 fallback하지 마라.
- 질문부터 하지 말고 저장소를 직접 조사하라.
- 기존 변경사항을 reset, clean, checkout, stash 또는 삭제하지 마라.
- preflight 생성, plan, setup, verify 단계에서는 자동 commit/push를 하지 마라.
- 사용자가 `cycle`을 직접 실행했고 설정에서 리뷰 사이클을 활성화한 경우에만
  task branch, commit, push, Draft PR, 선택된 merge 작업을 수행하라.
- secret을 생성하거나 출력하지 마라.
- 전체 script를 sudo로 실행하지 마라.
- package.json에 선언되지 않은 dependency를 임의로 설치하지 마라.
- 제품 기능이나 tasks.yaml의 completed 상태를 직접 수정하지 마라.

다음 파일을 조사하라:

- tasks.yaml
- PRODUCT.md, AGENTS.md
- ralph.workflow.json
- .ralphy/config.yaml
- package.json과 lockfile
- package scripts
- tsconfig, lint, test, build, Playwright 설정
- Dockerfile, compose 파일
- .env.example 계열
- CI workflow와 README

`ralph.workflow.json`이 없거나 스키마가 잘못되었으면 추측하지 말고 BLOCKER로 보고하라.

package script는 재귀적으로 분석하라.

예:
`npm run check`
→ typecheck, lint, test, build, test:e2e
→ tsc, eslint, vitest, next, playwright
→ 각 command를 제공하는 dependency가 package.json에 선언되어 있는지 확인

특히 다음 오류를 구분하라:

1. dependency는 선언되어 있지만 install되지 않음
2. script가 command를 사용하지만 dependency가 선언되지 않음
3. devDependency가 제외된 상태로 설치됨
4. package.json과 lockfile 불일치
5. runtime 또는 package manager 버전 불일치
6. browser 또는 OS library 누락
7. 환경변수, 인증, port 또는 Docker 권한 문제
8. Codex 모델 또는 추론 강도 override가 현재 CLI에서 거부됨
9. 리뷰 사이클에 필요한 Git remote, GitHub CLI 인증 또는 PR 권한 누락

`tsc: not found` 처리 규칙:

- package.json에 `typescript`가 선언되어 있다면 lockfile 기반으로 설치한다.
  npm + package-lock.json이면 `npm ci --include=dev`를 사용한다.
- 설치 후 `node_modules/.bin/tsc` 존재 여부를 검증한다.
- `typescript`가 선언되어 있지 않다면 global install이나
  임의의 `npm install -D typescript`를 실행하지 마라.
- 이 경우 seed manifest BLOCKER로 보고하고 중단하라.

다음 파일을 생성하라:

1. `ralph.environment.json`
   - runtime 최소 버전
   - package manager
   - 필수 command와 Codex CLI
   - 필수 환경변수 이름
   - port와 service
   - Docker/Playwright 필요 여부
   - quality command
   - 사용자 승인이 필요한 권한 작업
   - `ralph.workflow.json`의 schema version과 SHA-256 fingerprint
   - 구현 및 리뷰 단계의 model/reasoning override 적용 여부

2. `scripts/ralph-preflight.sh`

지원 명령:

```bash
./scripts/ralph-preflight.sh plan tasks.yaml
./scripts/ralph-preflight.sh setup tasks.yaml
./scripts/ralph-preflight.sh verify tasks.yaml
./scripts/ralph-preflight.sh run tasks.yaml -- <ralphy options>
./scripts/ralph-preflight.sh cycle tasks.yaml -- <ralphy options>
```

스크립트 요구사항:

- Bash strict mode 사용: `set -Eeuo pipefail`
- Git root를 자동 탐색
- idempotent하게 반복 실행 가능
- 경로와 argument를 안전하게 quote
- 명령 인자는 Bash array로 조립하고 `eval` 사용 금지
- 실패 시 non-zero exit
- 실패 원인과 사용자가 실행할 해결 명령 출력
- runtime artifact는 `.ralphy/preflight/` 아래에 저장
- `.ralphy/preflight/`가 Git 추적 대상이 되지 않도록 확인하고 아니면 경고
- root로 실행하면 거부
- secret 값은 출력하지 않음
- interrupt 또는 실패 시 현재 branch와 PR을 보존하고 재실행 가능한 상태 파일을 남김
- 다른 작업 트리의 사용자 변경을 stash, reset, checkout, clean 또는 삭제하지 않음

각 명령의 역할:

### plan

- 설치나 시스템 변경 없이 정적 분석만 수행
- 필요한 설치, 환경변수, port, service, 권한을 보고
- manifest, lockfile, workflow schema blocker 탐지
- 리뷰 사이클 활성화 시 GitHub remote, `gh`, PR과 merge 요구사항을 보고
- `.ralphy/preflight/report.md` 생성
- blocker가 있으면 non-zero 종료

### setup

- 먼저 plan 수행
- blocker가 없을 때만 deterministic install 수행
- npm + package-lock.json: `npm ci --include=dev`
- pnpm/yarn/bun은 해당 lockfile의 frozen install 사용
- manifest와 lockfile을 임의로 변경하지 않음
- 필요한 Playwright browser는 local dependency가 있을 때만 설치
- sudo가 필요한 명령은 자동 실행하지 말고
  `.ralphy/preflight/privileged-actions.sh`에 작성
- secret 입력이나 browser login은 사용자 작업으로 남김

### verify

다음을 검증하라:

- Git 저장소와 쓰기 권한
- runtime/package manager 버전
- Ralphy와 Codex CLI command
- `codex login status`
- `ralph.workflow.json`의 고정 스키마와 허용 값
- manifest/lockfile 일관성
- project dependency 설치 상태
- tsc, eslint, vitest, next, playwright 등 local executable
- 필수 환경변수 이름의 값 존재 여부
- port 가용성
- Docker daemon/socket 접근
- Playwright browser
- Git user.name/user.email
- .ralphy/config.yaml
- 가능한 경우 Ralphy dry-run
- 프로젝트 quality command
- `git diff --check`

모델 설정 검증:

- 구현 model이 `inherit`가 아니면 Ralphy의 `--model` 인자로 전달 가능한지 확인
- 구현 reasoning_effort가 `inherit`가 아니면 Codex engine argument의
  `-c model_reasoning_effort=<value>`로 전달 가능하게 구성
- 리뷰 model 또는 reasoning_effort가 `inherit`이면 구현 단계의 유효 설정을 상속
- 사용자 지정 override가 있으면 `codex exec --ephemeral --sandbox read-only`의
  최소 smoke invocation으로 조합을 검증
- smoke invocation은 저장소를 수정하지 않고 비밀값을 출력하지 않으며 fallback하지 않음
- 계정, 모델 접근 또는 호환성 오류는 USER ACTION REQUIRED 또는 BLOCKED로 보고

리뷰 사이클 활성화 시 추가 검증:

- `gh --version`과 `gh auth status`
- push 가능한 `origin`과 기본 branch
- 현재 사용자의 저장소 push/PR 권한
- working tree가 clean인지 여부
- PR 생성과 선택한 merge mode에 필요한 저장소 설정

quality command는 저장소에서 분석하여 결정하라.
`npm run check`가 정의되어 있다면 반드시 실행하라.

검증 성공 시:

- `.ralphy/preflight/PASSED` 생성
- 검증 시각과 environment fingerprint 저장

다음 파일이 변경되면 기존 PASS를 무효화하라:

- tasks.yaml
- package.json과 lockfile
- .ralphy/config.yaml
- ralph.workflow.json
- ralph.environment.json
- 주요 test/build 설정

### run

- 실행 직전에 verify를 다시 수행
- verify가 성공하고 fingerprint가 최신일 때만 Ralphy 실행
- Ralphy를 반드시 `ralphy --codex`로 실행
- `ralph.workflow.json`의 구현 model/reasoning 설정을 안전한 인자 배열로 적용
- `--` 뒤 사용자가 제공한 argument를 배열로 그대로 전달
- 리뷰 사이클을 시작하지 않음
- Ralphy exit code를 그대로 반환

### cycle

Producer-Reviewer 패턴의 bounded loop로 구현하라.

- `review_cycle.enabled`가 `true`일 때만 실행하고 아니면 non-zero로 종료하며 `run` 명령을 안내
- 매 시작과 재개 시 verify와 environment fingerprint를 다시 확인
- 상태를 `.ralphy/preflight/cycle-state.json`에 기록하고 GitHub의 branch/PR 상태와 대조
- 동시에 정확히 하나의 task branch와 하나의 PR만 다룸
- Ralphy를 `--max-iterations 1`, `--branch-per-task`, `--create-pr`,
  `--draft-pr`, 명시된 base branch로 실행
- Ralphy가 선택한 정확히 한 태스크의 quality command, task completion update,
  progress evidence와 단일 task commit을 확인
- PR의 head SHA와 로컬 HEAD가 일치하는지 확인

리뷰 단계:

- 구현 세션과 분리된 새 `codex exec review` 프로세스를 사용
- 리뷰 model/reasoning 설정을 적용하고 `--ephemeral`, read-only sandbox를 사용
- base branch 기준 전체 PR diff를 검토
- 결과를 고정 JSON schema로 받아
  `.ralphy/preflight/reviews/<task-key>/<head-sha>.json`에 보존
- 결과는 `pass` 또는 `changes_required`, blocking findings와 non-blocking findings를 구분
- malformed, missing 또는 ambiguous 결과는 통과로 취급하지 않고 fail closed

수정 단계:

- blocking finding이 있으면 리뷰와 분리된 새 Codex 프로세스를 workspace-write로 실행
- 리뷰 artifact, 원래 task acceptance criteria와 현재 diff만 입력으로 제공
- unrelated refactor, task 변경, protected file 변경을 금지
- 수정 후 task validation, quality command와 `git diff --check`를 다시 실행
- task당 정확히 한 커밋을 유지하도록 기존 task commit을 amend하고
  `git push --force-with-lease`로 같은 PR branch를 갱신
- 갱신된 head SHA를 대상으로 이전 세션을 재사용하지 않고 새 리뷰를 실행
- `max_cycles`를 넘기면 Draft PR과 branch를 보존하고 BLOCKED로 종료

merge와 다음 태스크:

- 최신 head SHA의 리뷰가 `pass`이고 blocking finding이 0개인지 확인
- 최신 head SHA에서 quality command가 성공했는지 확인
- GitHub에 설정된 status check가 있으면 전부 성공할 때까지 기다리고 실패하면 중단
- PR이 mergeable이고 base branch가 예상과 일치하는지 확인
- `merge_mode=manual`이면 PR을 ready로 전환하고 중단한다. 사용자가 merge한 뒤 재실행하면 다음 태스크로 진행
- `merge_mode=automatic`이면 위 gate를 모두 통과한 경우에만 PR을 squash merge하고
  GitHub에서 merged 상태와 merge commit을 다시 조회해 확인
- merge 확인 후에만 base branch를 `git pull --ff-only`로 동기화하고 다음 incomplete task를 시작
- incomplete task가 없으면 성공 종료
- closed/unmerged PR, conflict, stale review SHA, failed check 또는 GitHub 상태 불일치가 있으면 자동 진행하지 않음

Pilot 예시:

```bash
./scripts/ralph-preflight.sh run tasks.yaml -- \
  --yaml tasks.yaml \
  --max-iterations 1 \
  --max-retries 1 \
  --no-browser
```

리뷰 사이클 예시:

```bash
./scripts/ralph-preflight.sh cycle tasks.yaml -- \
  --yaml tasks.yaml \
  --max-retries 1 \
  --no-browser
```

작업 순서:

1. 저장소, tasks.yaml과 ralph.workflow.json 분석
2. environment contract 생성
3. preflight script 생성
4. shell syntax 검증
5. plan 실행
6. blocker가 없으면 setup 실행
7. verify 실행
8. 자동 수정 가능한 문제만 수정하고 verify 재실행
9. sudo, secret, 인증 등 사용자 작업이 남으면 Ralphy를 실행하지 않고 중단
10. 모든 검증이 통과하면 선택된 workflow에 맞는 pilot 명령만 출력

최종 응답 형식:

## Preflight 상태

PASS / BLOCKED / USER ACTION REQUIRED

## 감지한 환경

runtime, package manager, quality command, Ralphy engine (Codex),
구현 model/reasoning, 리뷰 사이클과 merge mode

## 수행한 작업

실제로 실행한 설치와 검증

## 사용자 작업

필요한 명령과 그 이유

## 검증 결과

dependencies, executables, env, ports, Docker/Playwright,
workflow schema, Codex overrides, GitHub prerequisites,
Ralphy dry-run, quality command, git diff check

## 다음 명령

BLOCKED이면 verify 재실행 명령,
PASS이면 preflight wrapper를 이용한 run 또는 cycle 명령

파일만 작성하고 끝내지 마라.
현재 환경에서 가능한 plan, setup, verify를 직접 실행하라.
검증하지 않은 항목을 성공했다고 표현하지 마라.
모든 검증이 통과하기 전에는 실제 feature task를 실행하지 마라.

이제 작업을 시작하라.
