---
title: K8s Pod에 Time Zone 설정하기
author: garit
date: 2022-11-20 11:30:00 +0900
categories: [Kubernetes, Cases]
tags: [k8s]
render_with_liquid: false
---

## K8s Pod에 Time Zone 설정하기

### 상황
생성된 Pod의 로그를 kubectl logs 명령어로 확인하면 로그가 찍히는 시간이 현재 시간과 달라 에러를 디버깅할 때 어려움이 생겼다.  
<br/>
그래서 Pod에 접속하여 /etc/localtime 을 확인해보니 UTC로 되어있었다.  
<br/>

### 해결방안

**Pod에 Time Zone을 설정해서 한국 표준 시간으로 맞춰야 한다.**    
<br/>

Time Zone을 설정하는 방법은 여러 개가 있다.  
<br/>

**1. TZ 환경변수 설정하기**  
<br/>

ConfigMap을 이용하여 환경변수에 KST를 설정해준다.  
<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test
  name: test-deploy
  namespace: test-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - image: 123123.dkr.ecr.ap-northeast-2.amazonaws.com/test-ecr:0.1.2
        imagePullPolicy: Always
        name: test-container
        ports:
        - containerPort: 8080
          name: p-8080
        envFrom:
        - configMapRef:
            name: test-configmap
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-configmap
  namespace: test-ns
data:
  TZ: Asia/Seoul
---
```

위와 같이 TZ을 Asia/Seoul로 설정했으나 적용이 되지 않는다. 그래서 다음 방법으로 해결했다.  
<br/>

**2.노드의 zoneinfo를 마운트하여 localtime을 설정하기**  
<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test
  name: test-deploy
  namespace: test-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - image: 123123.dkr.ecr.ap-northeast-2.amazonaws.com/test-ecr:0.1.2
        imagePullPolicy: Always
        name: test-container
        ports:
        - containerPort: 8080
          name: p-8080
        envFrom:
        - configMapRef:
            name: test-configmap
        volumeMounts:
        - name: tz-config
          mountPath: /etc/localtime
      volumes:
      - name: tz-config
        hostPath:
          path: /usr/share/zoneinfo/Asia/Seoul
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-configmap
  namespace: test-ns
data:
  TZ: Asia/Seoul
```
<br/>

위와 같은 방법으로 설정하니 Time Zone이 KST로 잘 적용되었다.  
<br/>

- 어떤 경우에는 base image에 /usr/share/zoneinfo가 없어서 해당 directory를 같이 마운트해줘야 KST가 적용되는 경우도 있다.  
<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test
  name: test-deploy
  namespace: test-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - image: 123123.dkr.ecr.ap-northeast-2.amazonaws.com/test-ecr:0.1.2
        imagePullPolicy: Always
        name: test-container
        ports:
        - containerPort: 8080
          name: p-8080
        envFrom:
        - configMapRef:
            name: test-configmap
        volumeMounts:
        - name: tz-data
          mountPath: /usr/share/zoneinfo
        - name: tz-config
          mountPath: /etc/localtime		  
      volumes:
      - name: tz-data
        hostPath:
          path: /usr/share/zoneinfo
      - name: tz-config
        hostPath:
          path: /usr/share/zoneinfo/Asia/Seoul		  
---

<br/>


그러나 혹시 spring 어플리케이션에서 위와 같은 방법으로도 적용이 안 된다면 JAVA 실행 옵션에 아래와 같이 설정해보자.

**3. JVM 실행 옵션 주기**


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test
  name: test-deploy
  namespace: test-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - image: 123123.dkr.ecr.ap-northeast-2.amazonaws.com/test-ecr:0.1.1
        imagePullPolicy: Always
        name: test-container
        ports:
        - containerPort: 8080
          name: p-8080
        envFrom:
        - configMapRef:
            name: test-configmap
        volumeMounts:
        - name: tz-config
          mountPath: /etc/localtime
      volumes:
      - name: tz-config
        hostPath:
          path: /usr/share/zoneinfo/Asia/Seoul
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-configmap
  namespace: test-ns
data:
  TZ: Asia/Seoul
  JAVA_OPT: "-Dspring.profiles.active=dev -Duser.timezone=Asia/Seoul"
```

위 ConfigMap에서 설정한 -Duser.timezone=Asia/Seoul는 Dockerfile의 entrypoint에서 java -jar ${JAVA_OPT}로 실행된다.  
<br/>

세 가지 방법을 통해 로그가 KST로 나오는 것을 확인했다.  

