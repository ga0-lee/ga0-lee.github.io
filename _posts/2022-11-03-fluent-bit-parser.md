---
title: Fluent-bit의 parser에 사용된 정규식 파악하기
author: garit
date: 2022-11-03 10:00:00 +0900
categories: [Kubernetes, Cases]
tags: [k8s]
render_with_liquid: false
---

## Fluent-bit의 parser에 사용된 정규식 파악하기

### Fluent-bit의 parser란

Parser는 ...
[https://docs.fluentbit.io/manual/pipeline/filters/parser](https://docs.fluentbit.io/manual/pipeline/filters/parser)


### Fluent-bit parser에 적용된 정규식 파헤쳐보기

```bash
[PARSER]
    Name multiline
    Format regex
    Regex /(?<time>Dec \d+ \d+\:\d+\:\d+)(?<message>.*)/
    Time_Key  time
    Time_Format %b %d %H:%M:%S

```





참고
- [https://idreamtbest.tistory.com/71](https://idreamtbest.tistory.com/71)
- [http://sweeper.egloos.com/3064808](http://sweeper.egloos.com/3064808)
