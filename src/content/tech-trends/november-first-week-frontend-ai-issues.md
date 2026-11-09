---
title: "2026년 11월 첫째 주 프론트엔드 & AI 이슈"
description: "AI 역할 분리, Agent 의존성 관리, CI/CD 보안, 정적 분석과 인증 보안 흐름을 살펴본다."
date: "2026-11-05"
tags:
  - AI Agent
  - Frontend Security
  - GitHub Actions
  - CodeQL
  - Performance
---

# 2026년 11월 첫째 주 프론트엔드 & AI 이슈

2026년 10월 30일부터 11월 5일까지 국내외 기술 블로그와 공식 개발 블로그에서 확인한 내용을 바탕으로, 프론트엔드 개발자에게 도움이 되는 이슈 여섯 가지를 기록한다.

이번 주에는 AI가 잘하는 작업과 일반 코드가 처리해야 하는 작업을 분리하는 방법, AI Agent의 의존성과 자동화 권한을 관리하는 방법, 테스트와 검증의 중요성을 중심으로 살펴봤다.

## 1. LINE: LLM에게 계산까지 맡기지 않는 설계

- 발표일: 2026.10.27
- 출처: LY Corporation Tech Blog
- 키워드: AI Agent, Tool Calling, Data Validation

LY Corporation은 광고 데이터를 분석해 리포트를 생성하는 AI Agent를 만들었다. 초기에는 매출 데이터와 비즈니스 정보를 LLM에 전달해 리포트 전체를 생성했지만, 수치 계산 오류와 데이터 누락, 실행할 때마다 달라지는 형식, 근거가 부족한 원인 분석 문제가 발생했다.

이후 정확한 수치 계산과 주요 항목 선정은 일반 코드가 담당하고, LLM은 검증된 데이터를 바탕으로 자연어 설명을 생성하도록 역할을 분리했다. 추가 분석이 필요할 때만 AI Agent가 Tool Calling으로 데이터를 조회하고, 조회 범위와 반복 횟수는 시스템에서 제한했다.

Money Log에 AI 소비 분석 기능을 추가한다면 같은 구조를 적용할 수 있다.

```text
거래 데이터
    ↓
JavaScript로 합계와 증감률 계산
    ↓
검증된 분석 데이터
    ↓
LLM으로 소비 패턴 설명 생성
    ↓
사용자에게 인사이트 제공
```

합계나 증감률처럼 정확성이 필요한 계산은 코드로 처리하고, 소비 패턴을 설명하는 문장처럼 해석이 필요한 영역을 LLM에 맡기면 결과를 검증하기 쉬워진다.

## 2. GitHub: AI Agent의 Skill과 프롬프트도 의존성으로 관리

- 발표일: 2026.11.03
- 출처: GitHub Changelog
- 키워드: AI Agent, Dependabot, Dependency Management

GitHub Dependabot은 Agent Package Manager를 지원하기 시작했다. AI Agent에서 사용하는 Skill, 프롬프트, 지침 파일도 버전 업데이트 대상으로 관리할 수 있다.

`apm.yml`에는 의존성을 정의하고 `apm.lock.yaml`에는 사용할 버전을 고정한다. Dependabot은 새 버전이 나오면 Pull Request를 만들어 업데이트를 제안한다. 현재는 버전 업데이트를 지원하며, 보안 업데이트는 지원하지 않는다.

기존에는 npm 패키지처럼 실행 코드의 의존성을 관리하는 일이 중심이었다. 이제 AI Agent가 개발 과정에 사용되면서 프롬프트와 Skill도 결과에 영향을 주는 의존성이 된다. 여러 프로젝트에서 공통 Skill을 재사용한다면, 규칙 변경이 기존 프로젝트에 미치는 영향을 확인할 필요가 있다.

## 3. GitHub Actions: 위험한 PR 이벤트 실행 기본 차단

- 발표일: 2026.11.02
- 출처: GitHub Changelog
- 키워드: GitHub Actions, CI/CD, Security

GitHub는 공개 저장소에서 `pull_request_target` 이벤트를 기본으로 차단하는 정책을 발표했다. 2026년 12월 2일부터 적용될 예정이며, 사전에 허용 정책을 설정한 워크플로만 예외적으로 실행할 수 있다.

`pull_request_target`은 Pull Request를 받는 원본 저장소의 권한으로 실행된다. 외부 기여자가 올린 신뢰할 수 없는 코드를 실행하면 저장소의 비밀 정보가 노출될 위험이 있다.

일반적인 프론트엔드 프로젝트에서는 Pull Request 생성 시 TypeScript 검사, ESLint, 테스트를 실행하는 흐름부터 구성할 수 있다. 자동화는 편리하지만, 어떤 이벤트가 어떤 권한으로 실행되는지 함께 확인해야 한다.

## 4. GitHub: CodeQL 정적 분석 정확도 개선

- 발표일: 2026.11.04
- 출처: GitHub Changelog
- 키워드: CodeQL, Static Analysis, JavaScript, TypeScript

GitHub는 CodeQL 2.27.4를 공개했다. CodeQL은 소스 코드를 실행하지 않고 분석해 잠재적인 보안 취약점을 찾는 정적 분석 도구이다. 이번 업데이트에서는 여러 언어의 분석 정확도를 개선하고, 일부 잘못된 경고를 줄였다.

JavaScript와 TypeScript 관련해서는 정규표현식 기반 취약점 분석에 사용되는 일부 CWE 메타데이터가 수정됐다. 보안 분석 도구는 실제로 위험하지 않은 코드를 경고할 수도 있으므로, 경고의 개수뿐 아니라 정확성과 맥락을 함께 확인해야 한다.

각 도구는 서로 다른 문제를 다룬다.

| 도구 | 주요 역할 |
| --- | --- |
| TypeScript | 타입 오류 검사 |
| ESLint | 코드 품질과 규칙 검사 |
| CodeQL | 잠재적인 보안 취약점 분석 |
| Playwright | 실제 사용자 동작 검증 |

이 도구들을 함께 사용하면 타입, 코드 품질, 보안, 사용자 흐름처럼 서로 다른 종류의 문제를 발견할 수 있다.

## 5. GitHub Desktop: 수명이 짧은 인증 토큰

- 발표일: 2026.11.02
- 출처: GitHub Changelog
- 키워드: Authentication, Token, Security

GitHub Desktop에는 수명이 짧은 인증 토큰을 사용하는 방식이 추가됐다. 장기간 유효한 인증 정보가 유출되면 공격자가 이를 오랫동안 악용할 수 있다. 반면 수명이 짧은 토큰은 유출되더라도 악용할 수 있는 시간을 줄이는 데 도움이 된다.

React에서 로그인 기능을 구현할 때도 토큰을 어디에 저장할지뿐 아니라, 얼마나 오래 유효한지와 어떻게 갱신하는지를 함께 이해해야 한다.

- Access Token과 Refresh Token의 역할
- 토큰 만료 시간과 갱신 흐름
- `HttpOnly` Cookie 사용 이유
- 인증과 인가의 차이

## 6. LINE: AI 보안 Agent에도 사람의 승인 남기기

- 발표일: 2026.10.23
- 출처: LY Corporation Tech Blog
- 키워드: AI Agent, Security Automation, Human-in-the-loop

LY Corporation은 보안 업무를 지원하는 AI Agent 플랫폼 SAGE의 개발 과정을 공유했다. 이 사례는 AI Agent가 보안 업무를 자동화하더라도 사람이 검토하고 판단하는 단계를 남겨두는 **Human-in-the-loop** 방식을 다룬다.

AI는 반복적인 정보 수집과 분석을 지원하고, 중요한 판단과 승인에는 사람이 개입한다. AI Agent가 실제 시스템과 더 많이 연결될수록 잘못된 판단의 영향도 커지므로, 위험도에 따라 자동 실행과 사람의 승인을 구분해야 한다.

AI 코딩 도구를 사용할 때도 같은 흐름을 적용할 수 있다.

```text
AI 코드 생성
    ↓
TypeScript와 ESLint 검사
    ↓
테스트 실행
    ↓
변경 사항 검토
    ↓
사람의 최종 승인
```

## React와 Next.js 업데이트

이번 주 공식 블로그에서는 새로운 React 주요 릴리스나 Next.js 주요 버전 발표를 확인하지 못했다. 새로운 기능을 따라가기보다 기존 React 기능과 성능, 테스트 개념을 학습하는 편이 더 실용적인 시기이다.

Next.js에서는 AI Agent 지원, 개발 환경 디버깅, 서버 렌더링 성능 개선 흐름이 계속 강조되고 있다. 프레임워크 기능뿐 아니라 검증과 성능을 포함한 개발 과정 전체를 함께 이해할 필요가 있다.

## 이번 주 핵심 흐름

| 분야 | 주요 이슈 | 알아둘 개념 |
| --- | --- | --- |
| AI 서비스 | 계산과 해석의 역할 분리 | Tool Calling, Validation |
| AI 개발 | Agent 의존성 자동 업데이트 | Skills, Dependabot, Lockfile |
| CI/CD 보안 | PR 이벤트 권한 관리 | GitHub Actions, Secrets |
| 코드 품질 | CodeQL 분석 정확도 개선 | Static Analysis, SAST |
| 인증 보안 | 단기 인증 토큰 | OAuth, Token, Cookie Security |
| AI 보안 | 사람의 최종 승인 | Human-in-the-loop, Code Review |

## 이번 주 추천 학습

AI에게 기능 구현을 요청하기 전에 보안과 검증 기준을 먼저 작성하는 연습이 필요하다. 예를 들어 거래 CRUD 기능을 구현한다면 데이터 검증, 잘못된 입력 처리, 사용자별 데이터 접근 권한, 테스트 통과 조건을 먼저 정한다.

이렇게 하면 AI 활용 경험을 단순한 코드 생성이 아니라 요구사항 정의부터 검증까지 관리한 개발 경험으로 확장할 수 있다.

## 정리

- 정확성이 중요한 계산과 검증은 일반 코드가 담당하고, AI는 해석과 설명에 집중하도록 역할을 나눈다.
- AI Agent의 Skill과 프롬프트도 버전과 변경 이력을 관리하는 의존성이 된다.
- CI/CD 자동화에서는 이벤트와 실행 권한을 함께 검토한다.
- AI가 만든 결과는 타입 검사, 정적 분석, 테스트, 코드 리뷰를 거쳐 사람이 최종 승인한다.
