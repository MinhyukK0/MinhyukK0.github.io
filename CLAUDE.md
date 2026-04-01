# CLAUDE.md

## 프로젝트 개요

- Jekyll 기반 기술 블로그 (Chirpy 테마)
- URL: https://minhyukk0.github.io
- 언어: 한국어 (`lang: ko`)
- 타임존: Asia/Seoul

## 포스트 작성 규칙

### 파일 위치 및 네이밍

- 경로: `_posts/YYYY-MM-DD-<slug>.md`
- slug는 영문 소문자, 하이픈 구분

### Frontmatter 형식

```yaml
---
title: "제목"
date: YYYY-MM-DD HH:MM:SS +0900
categories: [카테고리명]
tags: [tag1, tag2, tag3]
---
```

### 카테고리

Chirpy 테마에서 카테고리는 포스트의 `categories` frontmatter로 자동 생성된다. 별도 카테고리 파일 불필요.

현재 사용 중인 카테고리:

- `Tech Exploration` — 기술 탐색, 아키텍처 분석
- `TroubleShooting` — 문제 해결 과정 기록

### 본문 스타일

- 한국어로 작성, 기술 용어는 영문 그대로 사용
- 경어체 아닌 평서체 사용 ("~다" 체)
- 개념 설명 → 코드 예시 → 비교표 순서로 구성
- ASCII 다이어그램 적극 활용
- 비교표(Markdown table)로 기술 간 차이점 정리

## 빌드

```bash
bundle install
bundle exec jekyll serve
```

## 설정

- `_config.yml` — 사이트 전역 설정
- `future: true` — 미래 날짜 포스트도 표시
- `toc: true` — 포스트에 목차 자동 생성
- `paginate: 10`
