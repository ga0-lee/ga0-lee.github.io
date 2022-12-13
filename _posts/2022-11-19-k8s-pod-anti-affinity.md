---
title: K8s PodAntiAffinity로 Pod의 중복 배포를 최소화 하기
author: garit
date: 2022-11-19 12:40:00 +0900
categories: [AWS, EKS]
tags: [aws, eks]
render_with_liquid: false
---

## K8s PodAntiAffinity로 Pod의 중복 배포를 최소화 하기

### 상황
생성된 Worker Node가 3개, Pod의 최소 replica 수도 3개인데 어떤 Deployment는 하나의 노드에 3개의 Pod가 전부 배포되어 있고, 어떤 Deployment는 1,2번에만 pod가 배포되어 있는 걸 발견했다.  
<br/>
이런 경우 어느 노드 또는 가용영역에 장애가 발생하게 되면 순간적으로 서비스 중단이 될 수 있기에 각 노드에 Pod가 골고루 배포되도록 설정할 필요가 있다.  
<br/>


