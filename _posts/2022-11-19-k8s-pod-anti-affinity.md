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

### 해결방안

**PodAntiAffinity를 사용하여 이미 같은 Deployment의 Pod가 배포된 노드에는 중복 배포가 되지 않도록 설정한다.**  
<br/>

podAntiAffinity 규칙은 스케줄러로 하여금 app=test 레이블을 가진 복수 개의 레플리카를 단일 노드에 배치하지 않게 한다. 이렇게 하여 Pod를 각 노드에 분산하여 생성한다.  
<br/>

affinity.podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution.weight 값에 따라 어느 Node에 배포할지 가중치를 줄 수 있다.  
<br/>

예를 들어 preferredDuringSchedulingIgnoredDuringExecution 규칙을 만족하는 노드가 2개 있고, 하나에는 app: test 레이블이 있고 다른 하나에는 type: nginx 레이블이 있으면, 스케줄러는 각 노드의 weight를 확인한 뒤 weight가 더 작은 쪽에 Pod를 배포한다.  
<br/>

즉, app: test 레이블을 가진 pod-A가 배포된 node-A와 type: nginx 레이블을 가진 pod-B가 배포된 node-B가 있을 때 아래의 test-deploy는 가중치가 더 작은 app: test 레이블을 가진 pod-A가 배포된 node-A에 배포된다.  
<br/>

또한 podAntiAffinity에서는 노드에 일관된 레이블을 지정해야 한다.  
<br/>

즉, 클러스터의 모든 노드는 topologyKey 와 매칭되는 적절한 레이블을 가지고 있어야 한다. 일부 또는 모든 노드에 지정된 topologyKey 레이블이 없는 경우에는 의도하지 않은 동작이 발생할 수 있다.  
<br/>

- 예시

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
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 10
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - test		
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: type
                  operator: In
                  values:
                  - nginx				  
              topologyKey: "kubernetes.io/hostname"
      containers:
      - image: 123123123.dkr.ecr.ap-northeast-2.amazonaws.com/test:0.1.1
        imagePullPolicy: Always
        name: test-container
        ports:
        - containerPort: 8080
          name: p-8080
        envFrom:
        - configMapRef:
            name: test-configmap
        resources:
          requests:
            memory: "1G"
            cpu: "1"
          limits:
            memory: "1G"
            cpu: "1"
        livenessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 60
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: tz-config
          mountPath: /etc/localtime
      volumes:
      - name: tz-config
        hostPath:
          path: /usr/share/zoneinfo/Asia/Seoul
```
