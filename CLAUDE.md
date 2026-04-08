# Devlog

Claude 세션 히스토리를 기술 블로그로 발행하는 레포지토리.

## 구조
- `_histories/`: `/log` 스킬이 쌓는 원본 히스토리 (로컬 전용, gitignore)
- `_posts/`: `/publish` 스킬이 생성하는 블로그 글
- `.claude/skills/publish/`: 발행 스킬

## 워크플로우
1. 작업 세션에서 `/log`로 히스토리 축적
2. 이 레포에서 `/publish`로 블로그 글 생성 및 배포
