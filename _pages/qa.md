---
title: "QA"
permalink: /qa/
layout: single
author_profile: true
toc: false
toc_sticky: false
comments: false
share: false
related: false
classes: wide
---

<p class="hub-lead">
  좋은 품질은 테스트를 많이 돌려서가 아니라, 무엇을 어떤 기준으로 검증할지 설계했을 때 나옵니다.
  테스트 전략과 자동화, 그리고 실패 원인을 끝까지 좇는 버그 분석을 이곳에 정리합니다.
</p>

<ul class="hub-topics">
  <li class="chip">테스트 전략</li>
  <li class="chip">테스트 자동화 · CI</li>
  <li class="chip">버그 리포트 · 분석</li>
  <li class="chip">API 테스트</li>
  <li class="chip">성능 · 부하 테스트</li>
  <li class="chip">QA 프로세스</li>
</ul>

<div class="hub-section" markdown="1">

## 테스트 전략 {#test-strategy}

요구사항을 검증 가능한 형태로 바꾸고, 위험도에 따라 테스트 범위를 정하는 방법을 다룹니다.
테스트 케이스 설계 기법(경계값·동등 분할·조합), 테스트 계획서와 완료 기준, 릴리스 전 점검표 같은 실무 단위의 글을 쌓아갑니다.

</div>

<div class="hub-section" markdown="1">

## 자동화 · CI {#automation}

한 번 만든 검증을 반복해서 돌릴 수 있게 만드는 일을 다룹니다.
Playwright · Selenium 기반 UI 테스트, API 자동화, GitHub Actions 파이프라인, 테스트가 깨지지 않게 유지하는 방법(flake 대응, 안정적인 셀렉터, 테스트 데이터 관리)을 정리합니다.

</div>

<div class="hub-section" markdown="1">

## 버그 분석 {#bug-analysis}

버그를 "재현된다/안 된다"로 끝내지 않고, 원인 후보를 좁혀가는 과정을 기록합니다.
로그와 재현 절차 정리, 최소 재현 케이스 만들기, 개발자와 공유할 수 있는 버그 리포트 작성법, 회귀를 막는 테스트로 남기는 방법을 다룹니다.

</div>

<div class="hub-section" markdown="1">

## QA 글 모아보기 {#qa-posts}

{% assign qa_posts = site.categories["QA"] %}
{% assign qa_count = qa_posts.size | default: 0 %}
{% if qa_count > 0 %}
<div class="entries-list">
  {% for post in qa_posts %}
    {% include archive-single.html type="list" %}
  {% endfor %}
</div>
{% else %}
<div class="hub-empty" markdown="1">
**아직 QA 카테고리로 발행된 글이 없습니다.**

새 글은 front matter에 `categories: [QA]` 를 넣으면 이 목록과 상단 **QA** 메뉴에 자동으로 모입니다.
그동안 쌓아둔 Unity · VR · 게임 기획 글은 [전체 아카이브]({{ '/categories/' | relative_url }})에서 볼 수 있습니다.
</div>
{% endif %}

</div>
