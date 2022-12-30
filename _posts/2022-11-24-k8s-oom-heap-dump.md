---
title: Java Pod에 OOM 발생시 Heap Dump 생성하는 설정하기
author: garit
date: 2022-11-24 11:00:00 +0900
categories: [Kubernetes, Cases]
tags: [k8s, java]
render_with_liquid: false
---

## Java Pod에 OOM 발생시 Heap Dump 생성하는 설정하기

### 상황

Java 기반의 Pod가 OOM이 발생하여 재생성 되었는데 이유가 무엇인지 확인할 수 없어 OOM killed 되었을 때 Heap Dump를 생성하는 옵션을 설정하고자 한다.  
<br/>


### 설정 방법

- Pod 및 ConfigMap 설정

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test
  name: test-deploy
  namespace: test-ns
...
spec:
...
      containers:
      - envFrom:
        - configMapRef:
            name: test-configmap	
        volumeMounts:
        - mountPath: /fs
          name: test-fs		
      volumes:
      - name: test-fs
        persistentVolumeClaim:
          claimName: test-pvc		  
...
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-configmap
  namespace: test-ns
data:
  JAVA_OPT: "-Dspring.profiles.active=dev -Duser.timezone=Asia/Seoul"
  HEAP_OPT: "-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/fs/heap_dump/${HOSTNAME}.hprof"		
                               # OOM 발생시 Heap Dump를 생성하겠다는 옵션   # Pod의 HOSTNAME은 Pod명이므로 Pod이름으로 해당 경로에 Heap dump 파일을 생성한다.
```

- entrypoint.sh 설정  

```bash
#!/bin/sh

java ${HEAP_OPT} -jar ${JAVA_OPT} /deploy/app.jar
```

위와 같이 entrypoint.sh에 java 실행 옵션으로 ConfigMap에 설정한 HEAP_OPT를 주면 Pod가 실행될 때 Heap dump를 생성하는 옵션이 적용된다.  
<br/>

이제 OOM killed가 발생했을 때 Pod에 마운트한 File system에 dump가 저장될 것이다.  
<br/> <br/>
    
