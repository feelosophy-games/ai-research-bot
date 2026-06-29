# AI 리서치 봇 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 주 1회(금 17:00 KST) AI 활용 사례·모델 뉴스를 웹 리서치해 "요약+인사이트" 마크다운 리포트를 전용 GitHub 저장소에 자동 누적하는 클라우드 봇을 구축한다.

**Architecture:** 봇 로직은 저장소 루트의 `research-prompt.md` 한 곳에 정의(단일 진실 공급원). 클라우드 루틴(Anthropic CCR)이 매주 이 저장소를 체크아웃해 프롬프트를 실행하고, `reports/YYYY-MM-DD.md`를 생성한 뒤 커밋·푸시한다. 사용자는 `git pull`로 결과를 받는다.

**Tech Stack:** Git + GitHub(`gh` CLI, 계정 `feelosophy-games`), Claude 클라우드 루틴(`schedule` 스킬 / `RemoteTrigger` 툴), 마크다운. 코드 빌드/패키지 없음 — 산출물은 프롬프트와 설정.

## Global Constraints

- 결과 저장소: `jp-projects/ai-research-bot`, GitHub 원격 `https://github.com/feelosophy-games/ai-research-bot`.
- 출력 파일 경로: `reports/YYYY-MM-DD.md` (실행 날짜, KST 기준).
- 실행 시각: 금요일 17:00 KST = **금요일 08:00 UTC** = cron `0 8 * * 5`. (루틴 cron은 UTC, 최소 1시간 간격)
- 리포트 주제: ① AI 모델/제품 뉴스, ② AI 활용/업무 자동화 사례 — 두 섹션 모두 포함.
- 리포트 항목 형식: `요약` / `왜 중요` / `업무 적용 포인트` 3요소 + 출처 링크.
- 알림(이메일/Slack/Notion), 항목 점수화, 과거 롤업은 범위 밖(YAGNI).
- 커밋 메시지에 attribution/`Co-Authored-By` 트레일러 금지(사용자 전역 규칙).

---

### Task 1: 저장소 스캐폴딩 + GitHub 원격 연결

저장소 구조와 README를 만들고 GitHub에 푸시해, 클라우드 루틴이 체크아웃할 수 있는 상태로 만든다.

**Files:**
- Create: `~/jp-projects/ai-research-bot/README.md`
- Create: `~/jp-projects/ai-research-bot/reports/.gitkeep`
- Existing: `.gitignore`(이미 존재), `docs/superpowers/specs/2026-06-29-ai-research-bot-design.md`(이미 존재)

**Interfaces:**
- Produces: GitHub 원격 저장소 `https://github.com/feelosophy-games/ai-research-bot` (public), 기본 브랜치 `main`에 초기 내용 푸시 완료.

- [ ] **Step 1: README 작성**

`~/jp-projects/ai-research-bot/README.md`:

```markdown
# AI 리서치 봇

주 1회(금 17:00 KST) AI 모델/제품 뉴스와 AI 활용·업무 자동화 사례를 리서치해
`reports/YYYY-MM-DD.md`로 누적하는 클라우드 봇.

## 구조
- `research-prompt.md` — 봇이 매주 실행하는 리서치 지시문(봇 로직의 단일 진실 공급원)
- `reports/` — 주차별 리포트 누적
- `docs/superpowers/` — 설계·구현 계획 문서

## 결과 받기
git pull 하면 최신 리포트가 `reports/`에 들어온다.

## 수동 실행(테스트)
저장소 루트에서:
claude -p "research-prompt.md 를 읽고 그 지시를 그대로 따라 이번 주 리포트를 생성하라"

## 스케줄 관리
클라우드 루틴 목록/수정: https://claude.ai/code/routines
```

- [ ] **Step 2: reports 디렉터리 유지 파일 생성**

빈 디렉터리는 git이 추적하지 않으므로 placeholder를 둔다.

Run: `touch ~/jp-projects/ai-research-bot/reports/.gitkeep`

- [ ] **Step 3: GitHub 저장소 생성 + 푸시**

Run:
```bash
cd ~/jp-projects/ai-research-bot
git add -A && git commit -m "저장소 스캐폴딩: README, reports 디렉터리"
gh repo create feelosophy-games/ai-research-bot --public --source=. --remote=origin --push
```
Expected: `gh`가 저장소를 생성하고 `main`을 푸시. 출력에 `https://github.com/feelosophy-games/ai-research-bot` 표시.

- [ ] **Step 4: 원격 확인**

Run: `cd ~/jp-projects/ai-research-bot && git remote -v && gh repo view feelosophy-games/ai-research-bot --json url -q .url`
Expected: origin이 `https://github.com/feelosophy-games/ai-research-bot`(.git) 으로 표시되고 repo URL이 출력됨.

---

### Task 2: 리서치 프롬프트 작성 + 로컬 드라이런 검증

봇 로직의 핵심인 `research-prompt.md`를 작성하고, 로컬에서 `claude -p`로 실제 실행해 형식에 맞는 리포트가 나오는지 검증한다. (이 프로젝트의 "테스트"는 드라이런 산출물이 형식 체크리스트를 통과하는지다.)

**Files:**
- Create: `~/jp-projects/ai-research-bot/research-prompt.md`
- Test(산출): `~/jp-projects/ai-research-bot/reports/<드라이런 날짜>.md`

**Interfaces:**
- Consumes: Task 1의 저장소 구조.
- Produces: `research-prompt.md` — 클라우드 루틴(Task 3)이 "읽고 그대로 따르는" 지시문. 다음을 보장: 출력 경로 `reports/YYYY-MM-DD.md`, 두 섹션 포함, 항목당 요약/왜 중요/업무 적용 포인트 + 링크, 끝에 `git add/commit/push`.

- [ ] **Step 1: research-prompt.md 작성**

`~/jp-projects/ai-research-bot/research-prompt.md`:

````markdown
# 리서치 봇 지시문

너는 이 저장소(ai-research-bot)에 체크아웃되어 실행 중인 리서치 에이전트다.
아래 절차를 그대로 수행해 이번 주 리포트를 만들고 커밋·푸시한다. 사람에게 질문하지 말고 끝까지 자동 수행한다.

## 0. 날짜 확정
- 셸에서 `date +%Y-%m-%d`(KST 환경이 아니면 UTC 날짜)를 구해 오늘 날짜 `TODAY`로 쓴다.
- 리서치 대상 기간은 "지난 7일"이다.

## 1. 리서치 주제 (둘 다 수집)
A. **AI 모델 / 제품 뉴스**: 신규 모델 출시, 주요 버전 업데이트, 주목할 제품/기능, 업계 동향.
B. **AI 활용 / 업무 자동화 사례**: 실무에 AI를 적용한 레퍼런스, 워크플로우, 생산성 도구 활용 사례.

## 2. 수집 방법
- WebSearch로 주제별 2~4개 쿼리를 던진다. 쿼리에 최신성을 넣는다(예: "AI model release this week", "AI workflow automation case study 2026").
- 유망 결과는 WebFetch로 본문을 확인해 사실을 검증한다(제목만 보고 쓰지 않는다).
- **필터링**: 지난 7일 밖의 오래된 자료 제외, 중복 출처 통합, 광고성/저품질 제외.
- 섹션당 3~6개 항목을 목표로 하되, 자료가 부족하면 수집된 만큼만 쓰고 그 사실을 명시한다(빈 리포트 금지).

## 3. 출력
- 파일: `reports/<TODAY>.md` (예: `reports/2026-07-03.md`). 같은 파일이 이미 있으면 덮어쓴다.
- 아래 형식을 정확히 따른다:

```markdown
# AI 리서치 리포트 — <TODAY> (지난 7일)

## 🗞️ AI 모델 / 제품 뉴스
### [<제목>](<링크>)
- **요약**: 무슨 일인지 2~3줄
- **왜 중요**: 인사이트 한두 줄
- **업무 적용 포인트**: 어떻게 써먹을지 한두 줄

(항목 반복)

## 🛠️ AI 활용 / 업무 자동화 사례
### [<제목>](<링크>)
- **요약**: ...
- **왜 중요**: ...
- **업무 적용 포인트**: ...

(항목 반복)

## 한 줄 정리
- 이번 주 핵심 3가지를 불릿으로.
```

## 4. 커밋 & 푸시
- `git add reports/<TODAY>.md`
- `git commit -m "리포트: <TODAY>"`  (커밋 메시지에 attribution/Co-Authored-By 트레일러를 넣지 않는다)
- `git push origin main`
- 푸시가 실패하면 에러 메시지를 그대로 출력하고 종료한다.
````

- [ ] **Step 2: 로컬 드라이런 실행**

저장소 루트에서 프롬프트를 실제 실행해 리포트를 생성한다.

Run:
```bash
cd ~/jp-projects/ai-research-bot
claude -p "research-prompt.md 를 읽고 그 지시를 그대로 따라 이번 주 리포트를 생성하라. 단, 이번에는 git push 는 하지 말고 파일 생성까지만 하라." \
  --allowedTools "Bash Read Write Edit WebSearch WebFetch" > ~/ai-research-dryrun.log 2>&1
echo "exit=$?"
```
Expected: `exit=0`, 그리고 `reports/`에 오늘 날짜 `.md` 파일이 생성됨.

> 주의(전역 규칙): 수 분 이상 걸릴 수 있으면 포그라운드 대기 금지. 길어지면 `nohup ... &`로 분리하고 로그를 나중에 Read.

- [ ] **Step 3: 산출물 형식 검증 (테스트 체크리스트)**

생성된 리포트가 형식을 만족하는지 확인한다.

Run:
```bash
cd ~/jp-projects/ai-research-bot
F=$(ls -t reports/*.md | head -1); echo "FILE=$F"
grep -q "## 🗞️ AI 모델 / 제품 뉴스" "$F" && echo "OK: 뉴스 섹션"
grep -q "## 🛠️ AI 활용 / 업무 자동화 사례" "$F" && echo "OK: 사례 섹션"
grep -q "## 한 줄 정리" "$F" && echo "OK: 정리 섹션"
grep -qE "\*\*업무 적용 포인트\*\*" "$F" && echo "OK: 인사이트 요소"
grep -qE "\]\(https?://" "$F" && echo "OK: 출처 링크"
```
Expected: 다섯 줄 모두 `OK:` 출력. 하나라도 빠지면 `research-prompt.md`를 수정하고 Step 2부터 반복.

- [ ] **Step 4: 프롬프트 커밋 (드라이런 산출물은 커밋하지 않음)**

Run:
```bash
cd ~/jp-projects/ai-research-bot
git add research-prompt.md README.md
git commit -m "리서치 프롬프트 추가 및 드라이런 검증"
git push origin main
```
Expected: 커밋·푸시 성공. (드라이런으로 만든 `reports/<오늘>.md`는 staging하지 않음 — 실제 주간 리포트는 루틴이 생성)

---

### Task 3: 주간 클라우드 루틴 등록 + 즉시 실행 검증 + 사용법 문서화

`schedule` 스킬(`RemoteTrigger`)로 매주 금 08:00 UTC 루틴을 만들고, "지금 실행"으로 실제 푸시까지 검증한 뒤 README에 루틴 ID/관리 링크를 적는다.

**Files:**
- Modify: `~/jp-projects/ai-research-bot/README.md` (루틴 ID·관리 링크 추가)

**Interfaces:**
- Consumes: Task 1의 GitHub repo URL, Task 2의 `research-prompt.md`(저장소에 푸시됨).
- Produces: 활성화된 클라우드 루틴 1개, 검증된 첫 리포트가 GitHub `main`에 푸시됨.

- [ ] **Step 1: 루틴 생성 (RemoteTrigger create)**

`ToolSearch select:RemoteTrigger`로 툴을 로드한 뒤 `action: "create"` 호출. body 핵심값:
- `name`: `"AI 주간 리서치"`
- `cron_expression`: `"0 8 * * 5"` (금 08:00 UTC = 금 17:00 KST)
- `enabled`: `true`
- `job_config.ccr`:
  - `environment_id`: `schedule` 스킬이 제시한 환경 id 중 하나(예: `env_01Sg754gqRQpyzjZaeu71i4r`) — 실행 시 최신 목록에서 확인해 사용
  - `session_context.model`: `"claude-sonnet-4-6"`
  - `session_context.sources`: `[{"git_repository": {"url": "https://github.com/feelosophy-games/ai-research-bot"}}]`
  - `session_context.allowed_tools`: `["Bash","Read","Write","Edit","Glob","Grep","WebSearch","WebFetch"]`
  - `events[0].data.uuid`: 새로 생성한 소문자 v4 UUID
  - `events[0].data.message.content` (프롬프트, 자기완결적):
    ```
    너는 ai-research-bot 저장소가 체크아웃된 클라우드 환경에서 실행 중이다.
    저장소 루트의 research-prompt.md 를 읽고 그 지시를 처음부터 끝까지 그대로 수행하라.
    리포트 파일을 생성하고 git add/commit/push 까지 완료하라. 사람에게 질문하지 말 것.
    ```

Expected: 응답에 routine ID 반환. 끝에 `https://claude.ai/code/routines/{ROUTINE_ID}` 링크 확보.

> 참고: cron은 UTC·최소 1시간 간격. 클라우드 에이전트는 로컬 파일 접근 불가 — 모든 컨텍스트가 저장소 안에 있어야 한다(그래서 프롬프트가 repo의 research-prompt.md를 읽는 구조).

- [ ] **Step 2: 즉시 실행으로 검증 (run now)**

`RemoteTrigger`를 `action: "run", trigger_id: "<ROUTINE_ID>"`로 호출해 한 번 돌린다.
클라우드 실행이므로 즉시 끝나지 않는다 — 잠시 후 GitHub에서 결과를 확인한다.

- [ ] **Step 3: 푸시 결과 확인**

수 분 뒤, 루틴이 실제로 리포트를 푸시했는지 확인한다.

Run:
```bash
cd ~/jp-projects/ai-research-bot
git pull --quiet origin main
ls -t reports/*.md | head -3
git log --oneline -3
```
Expected: 오늘 날짜의 새 `reports/*.md`가 들어오고, 로그에 `리포트: <오늘>` 커밋이 보인다.
- 만약 푸시가 안 됐다면: 루틴 상세(`action: "get"`)로 실행 로그/에러를 확인한다. 권한(클라우드 환경의 repo push 자격) 문제면, 저장소가 public이고 origin이 https인지 확인하고 프롬프트의 push 단계를 점검한다.

- [ ] **Step 4: README에 루틴 정보 추가 + 커밋**

`README.md`의 "스케줄 관리" 항목에 실제 routine ID와 링크를 추가한다.

```markdown
## 스케줄 관리
- 루틴: AI 주간 리서치 (매주 금 17:00 KST / 08:00 UTC)
- 관리: https://claude.ai/code/routines/<ROUTINE_ID>
- 목록/삭제: https://claude.ai/code/routines
```

Run:
```bash
cd ~/jp-projects/ai-research-bot
git add README.md
git commit -m "README에 클라우드 루틴 정보 추가"
git push origin main
```
Expected: 커밋·푸시 성공.

---

## Self-Review

**1. Spec coverage**
- 출력=마크다운 파일 → Task 2 형식 + Task 3 푸시 ✅
- 주 1회 금 17:00 KST 자동 → Task 3 cron `0 8 * * 5`(UTC 변환) ✅
- 클라우드 실행(PC off) → Task 3 CCR 루틴 ✅
- 새 전용 GitHub repo → Task 1 ✅
- 두 주제 + 요약/왜 중요/업무 적용 포인트 → Task 2 프롬프트·형식·검증 ✅
- 빈약한 주 처리, 중복/저품질 필터 → Task 2 프롬프트 2절 ✅
- 단독 수동 실행 가능(성공 기준) → Task 2 드라이런 + README ✅
- YAGNI(알림/점수화/롤업 제외) → 어느 태스크에도 없음 ✅

**2. Placeholder scan**
- `<ROUTINE_ID>`, `<TODAY>`, `<제목>` 등은 런타임에 채워지는 실제 변수 표시(플레이스홀더 금지 대상 아님). 모든 코드/명령은 구체값 제공 ✅

**3. Type/이름 일관성**
- 파일 경로 `reports/YYYY-MM-DD.md`, repo URL, cron `0 8 * * 5`, 섹션 제목 문자열이 Task 2 생성·검증·Task 3 프롬프트에서 동일하게 사용됨 ✅
- 검증 grep 문자열이 Task 2 출력 형식의 섹션 제목과 정확히 일치 ✅
