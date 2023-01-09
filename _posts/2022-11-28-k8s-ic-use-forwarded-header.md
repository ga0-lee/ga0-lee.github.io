---
title: Nginx Ingress Controller에서 Client IP 유지하게 설정하기
author: garit
date: 2022-11-28 10:00:00 +0900
categories: [Kubernetes, Cases]
tags: [k8s, ingress]
render_with_liquid: false
---

## Nginx Ingress Controller에서 Client IP 유지하게 설정하기

### 상황

http_x_forwarded_for 헤더 값으로 Client IP를 받아와 사용하는 로직이 있는 서비스가 있다.  
<br/>

그런데 Client IP가 Ingress Controller를 거치면서 http_x_forwarded_for 헤더 값이 Ingress Controller Pod IP로 변경되어 해당 서비스로 들어와 에러가 발생했다.  
<br/>

그래서 Client IP가 서비스 Pod까지 유지되게 설정을 해야 한다.

### 해결 방법

해당 헤더 값을 유지하려면 use-forwarded-header를 true로 변경해줘야 한다.  
<br/>

기본 값은 use-forwarded-header: "false"다.
<br/>

- 아래와 같이 ingress-controller pod가 사용하는 ConfigMap을 변경한다.  
> kubectl edit cm -n ingress-nginx ingress-nginx-controller  
<br/>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-controller
data:
  allow-snippet-annaotations: "true"
  # 새로 추가할 부분
  use-forwarded-headers: "true"
...  
```

ConfigMap을 수정하면 ingress-controller Pod에 바로 적용된다.  
<br/>

위와 같이 적용해주면 Ingress Controller를 거쳐도 Client의 IP가 서비스 Pod까지 유지되어 http_x_forwarded_for 헤더 값에 남게 된다.
<br/>

참고
- [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#use-forwarded-headers](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#use-forwarded-headers)
