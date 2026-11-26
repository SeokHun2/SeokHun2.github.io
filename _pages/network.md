---
title: "Network"
permalink: /network/
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
  "느린데요"에서 멈추지 않고, 어느 계층에서 무엇이 잘못됐는지 확인하는 과정을 정리합니다.
  TCP/IP와 HTTP의 동작 원리, 패킷을 직접 뜯어보는 분석, 그리고 실제 장애를 좁혀가는 트러블슈팅 기록입니다.
</p>

<ul class="hub-topics">
  <li class="chip">TCP/IP</li>
  <li class="chip">HTTP · HTTPS</li>
  <li class="chip">DNS</li>
  <li class="chip">패킷 분석</li>
  <li class="chip">Wireshark · tcpdump</li>
  <li class="chip">트러블슈팅</li>
</ul>

<div class="hub-section" markdown="1">

## TCP/IP & HTTP {#tcpip}

3-way handshake부터 혼잡 제어, TIME_WAIT, HTTP/1.1과 HTTP/2 · HTTP/3의 차이까지 —
"왜 이 요청이 이렇게 동작하는가"를 설명할 수 있도록 계층별로 정리합니다.
TLS 핸드셰이크와 인증서, DNS 조회 순서도 같은 맥락에서 다룹니다.

</div>

<div class="hub-section" markdown="1">

## 패킷 분석 {#packet}

Wireshark와 tcpdump로 실제 트래픽을 보고 읽는 연습을 기록합니다.
필터 문법, 재전송(retransmission)과 RST 패킷 해석, 지연이 발생하는 지점 찾기, 캡처 파일을 남기는 습관까지 실습 위주로 다룹니다.

</div>

<div class="hub-section" markdown="1">

## 트러블슈팅 {#troubleshooting}

증상에서 원인까지 좁혀가는 순서를 남깁니다.
연결은 되는데 응답이 없는 경우, 간헐적으로 끊기는 경우, 특정 환경에서만 느린 경우처럼 재현이 어려운 문제를 계층별로 확인하는 절차를 정리합니다.

</div>

<div class="hub-section" markdown="1">

## Network 글 모아보기 {#network-posts}

{% assign net_posts = site.categories["Network"] %}
{% assign net_count = net_posts.size | default: 0 %}
{% if net_count > 0 %}
<div class="entries-list">
  {% for post in net_posts %}
    {% include archive-single.html type="list" %}
  {% endfor %}
</div>
{% else %}
<div class="hub-empty" markdown="1">
**아직 Network 카테고리로 발행된 글이 없습니다.**

새 글은 front matter에 `categories: [Network]` 를 넣으면 이 목록과 상단 **Network** 메뉴에 자동으로 모입니다.
그동안 쌓아둔 Unity · VR · 게임 기획 글은 [전체 아카이브]({{ '/categories/' | relative_url }})에서 볼 수 있습니다.
</div>
{% endif %}

</div>
