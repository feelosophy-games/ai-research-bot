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
- 루틴: AI 주간 리서치 (매주 금 17:00 KST / 08:00 UTC)
- 루틴 ID: trig_01J3hMW68rjkaiDUDkr1oCap
- 관리: https://claude.ai/code/routines/trig_01J3hMW68rjkaiDUDkr1oCap
- 목록/삭제: https://claude.ai/code/routines
