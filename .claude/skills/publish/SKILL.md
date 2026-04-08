---
name: publish
description: "Use when the user wants to publish accumulated history files as blog posts to GitHub Pages. Triggers when: /publish, '발행해', '블로그 올려'"
---

# Devlog Publisher

축적된 히스토리 파일을 분석하여 기술 블로그 글로 가공하고 GitHub Pages에 배포한다.

## 실행 절차

### 1. 히스토리 스캔

`_histories/` 디렉토리를 스캔하여 미발행 히스토리 파일을 수집한다.

```bash
find _histories/ -name "*.md" -type f | sort
```

프로젝트/브랜치별로 그룹핑하여 사용자에게 목록을 보여준다:

```
발행 가능한 히스토리:

1. sleepthera / feature-connect-sentry (3개 파일, 2026-01-27 ~ 2026-01-29)
2. hosspie / feature-screen-publishing (1개 파일, 2026-02-15)

발행할 항목을 선택해주세요 (번호 또는 "all"):
```

### 2. 히스토리 읽기 및 분석

선택된 항목의 히스토리 파일들을 시간순으로 읽는다.
같은 프로젝트/브랜치의 히스토리가 여러 개면 하나의 여정으로 엮는다.

### 3. 문체 참조

`references/writing-style.md`를 읽고 문체 규칙을 숙지한다.

사용자의 Medium 블로그(https://medium.com/@kmjnnhyk)에서 최근 글 1-2개를 fetch하여 현재 문체를 확인한다. writing-style.md의 규칙과 실제 글의 톤을 모두 반영한다.

### 4. 블로그 글 작성

히스토리의 사고 과정을 서사적 흐름으로 풀어낸다.

**글 구조:**
- 도입: 어떤 상황에서 이 문제를 만났는지 (짧게)
- 전개: 사고 과정을 시간순으로 서술 (가설 → 검증 → 실패/성공의 흐름)
- 해결: 최종 판단과 근거
- (선택적) 배운 점이나 남은 과제

**Jekyll frontmatter:**
```yaml
---
layout: post
title: "(제목)"
date: YYYY-MM-DD
categories: [(project), (domain)]
tags: [(관련 기술 태그들)]
---
```

제목은 문제의 핵심을 담되, 사용자의 Medium 제목 스타일을 참고한다.
예: "생각보다 복잡한 Sentry 오프라인 캐싱", "오프라인에서 에러가 사라진다;;"

### 5. 보안 필터 적용

`references/security-filter.md`의 규칙에 따라 글을 검수한다.

- 실제 코드 → 개념/의사 코드로 대체
- 내부 파일 경로 → 일반화
- 기획/인프라 정보 → 제거 또는 일반화

### 6. 사용자 확인

초안을 사용자에게 보여주고 확인을 받는다.

```
---
초안 작성 완료. 보안 필터 적용됨.
아래 글을 확인하고 수정할 부분이 있으면 말씀해주세요.
---

(초안 전문)

---
발행할까요?
```

수정 요청이 있으면 반영 후 다시 확인.

### 7. 발행

```bash
# 커밋 및 푸시
cd ~/DEV/devlog
git add _posts/
git commit -m "post: (글 제목 요약)"
git push
```

### 8. 히스토리 정리

발행 완료된 히스토리 파일을 삭제한다.

```bash
# 발행된 히스토리 삭제
rm _histories/{project}/{branch}/{files}

# 빈 디렉토리 정리
find _histories/ -type d -empty -delete

# 삭제도 로컬에서 정리 (gitignore이므로 커밋 불필요)
```

## 핵심 원칙

- **서사로 풀어낸다** — 사고 과정을 나열이 아닌 흐름으로
- **사용자의 목소리** — Medium 블로그 톤을 따르고, AI 말투를 쓰지 않는다
- **보안 최우선** — 코드, 기획, 인프라 정보가 유출되지 않도록
- **사용자가 최종 결정** — 항상 초안 확인 후 발행
