# Engineering Notes

Cloud Infra & DevOps를 공부하고 실무에서 마주친 내용을 정리하는 개인 기술 블로그입니다.

사이트는 GitHub Pages와 Jekyll 기반의 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 테마로 운영합니다.

## Site

- URL: <https://ks-jeon.github.io>
- Language: Korean
- Theme: `jekyll-theme-chirpy`
- Main topics: Cloud, DevOps, Kubernetes, Observability, AI Agent

## Content

주요 글은 `_posts` 디렉터리에 작성합니다.

- AWS 계정/IAM, 3-Tier 아키텍처, 재해 복구 전략
- AWS와 Azure 간 Site-to-Site VPN 구성
- Kubernetes 학습 노트
- Datadog 기반 Observability, RUM, 데모 스크립트
- EC2, S3, .NET Core 애플리케이션 구성

프로필과 소개 페이지는 `_tabs/about.md`에서 관리합니다.

## Local Development

Ruby와 Bundler가 설치되어 있어야 합니다.

```bash
bundle install
bundle exec jekyll serve
```

로컬 서버가 실행되면 브라우저에서 <http://localhost:4000>으로 확인할 수 있습니다.

## Writing Notes

새 글은 `_posts/YYYY-MM-DD-title.md` 형식으로 추가합니다.

기본 front matter 예시는 다음과 같습니다.

```yaml
---
title: "Post Title"
date: 2026-01-01 09:00:00 +0900
categories: [Cloud, AWS]
tags: [aws, devops]
---
```

## Repository Structure

```text
.
├── _config.yml       # Site configuration
├── _posts/           # Blog posts
├── _tabs/            # About, categories, tags, archives pages
├── _data/            # Theme data files
├── _plugins/         # Jekyll plugins
├── assets/           # Styles and static assets
└── index.html
```

## License

이 저장소는 [MIT License](LICENSE)를 따릅니다.
