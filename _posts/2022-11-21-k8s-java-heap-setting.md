---
title: Java 기반 Pod에 Heap Memory 설정하기
author: garit
date: 2022-11-21 10:00:00 +0900
categories: [Kubernetes, Cases]
tags: [k8s, java]
render_with_liquid: false
---

## Java 기반 Pod에 Heap Memory 설정하기

### 상황

성능테스트를 하던 도중 원하는 목표치에 도달하기도 전에 OOM(Out of Memory) 에러가 발생하여 Java 기반의 Pod의 Heap 사이즈를 변경해달라는 요청을 받아 변경하려 한다.
<br/>

### 해결 방법

아래와 같이 java 실행 명령어에 HEAP_OPT 값을 주어 app.jar를 실행시켜야 한다.  
<br/>

> entrypoint.sh 
> java ${HEAP_OPT} -jar ${JAVA_OPT} app.jar

해당 HEAP_OPT를 ConfigMap에 설정하여 entrypoint.sh 가 실행될 때 같이 실행될 수 있도록 한다.  

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-configmap
  namespace: test-ns
data:
  TZ: Asia/Seoul
  JAVA_OPT: "-Dspring.profiles.active=dev -Duser.timezone=Asia/Seoul"
  HEAP_OPT: "-XX:MinRAMPercentage=50 -XX:MaxRAMPercentage=80 -XshowSettings:vm"
```

- 최소, 최대 Heap memory 값을 절대값으로 넣어주면 Pod의 Resource Limits 값이 달라질 때마다 그에 맞춰 변경을 해줘야하는 불편함이 있다. 그래서 MinRAMPercentage, MaxRAMPercentage 값을 통해 비율을 조정해주면 된다.  
<br/>

- 그리고 우리가 변경한 최소, 최대 Heap memory 사이즈가 제대로 적용됐는지 확인하기 위해 -XshowSettings:vm 값을 넣어준다. 그러면 Pod가 부팅될 때 로그에 세팅된 Heap 사이즈가 찍힌다.  
<br/>

> ex) Pod의 Resource Limits가 Memory 4G인 경우 Pod 실행을 위해 사용되는 memory 빼고 남는 memory 중 80인 약 3G 정도가 Heap Max 사이즈로 적용되었다.  

- 로그 예시  
![heap](https://user-images.githubusercontent.com/67899732/209062330-4a2c46bc-2f81-43e4-bc25-6cd267bdfd67.png)


