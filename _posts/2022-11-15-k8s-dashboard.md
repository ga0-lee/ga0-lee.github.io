---
title: EKS에 Kubernetes Dashboard 배포하기
author: garit
date: 2022-11-15 10:00:00 +0900
categories: [Kubernetes, Cases]
tags: [k8s, eks]
render_with_liquid: false
---

## EKS에 Kubernetes Dashboard 배포하기

### 상황

AWS EKS에 Kubernetes Dashboard를 배포하여 관리하고자 한다.  
<br/>

현 사이트의 경우 보안상의 이유로 Proxy를 통해 Kubernetes Dashboard에 접속해야 한다.  
<br/>

그래서 아래와 같은 경로를 통해 Kubernetes Dashboard Pod에 도달하게 된다.  

> PC -> Proxy Server(Nginx) -> Ingress Controller -> Ingress -> Kubernetes Dashboard Service -> Kubernetes Dashboard Pod
<br/>

Kubernetes Dashboard는 무조건 https로 접속해야 하는데 앞쪽의 Ingress 또는 Proxy server 등에서 ssl 처리를 해주면 Kubernetes Dashboard 자체는 http로 설정이 가능하다.  
<br/>

그래서 나는 제일 앞단에 있는 Nginx에 ssl 설정을 해줄 것이다.

### Kubernetes Dashboard 배포하기

먼저 Kubernetes Dashboard를 배포해보자.  
<br/>

**1.Kubernetes Dashboard yaml 파일을 다운받는다.**  

> wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml -O k8s-dashboard.yaml

**2.아래와 같이 포트 및 옵션을 수정하여 배포한다.**

- 포트는 편의를 위해 모두 8001로 변경한다.  
- 아래의 예시는 수정할 부분만 가져온 것이다.  

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8001                  # 포트 수정
      targetPort: 8001            # 포트 수정
  selector:
    k8s-app: kubernetes-dashboard
---
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.7.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8001                 # 포트 수정
              protocol: TCP
          args:
            #- --auto-generate-certificates       # 주석 처리
            - --namespace=kubernetes-dashboard    # namespace를 설정한다
			- --insecure-bind-address=0.0.0.0     # 모든 http 접속을 허용한다 (추가)
            - --enable-insecure-login             # http 로그인 허용한다 (추가)
            - --token-ttl=10800	# 토큰 세션 유지시간 - 선택 사항임
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTP   # http로 변경
              path: /
              port: 8001     # 포트 수정
            initialDelaySeconds: 30
            timeoutSeconds: 30
```

위와 같이 변경한 후 배포한다.

> kubectl apply -f k8s-dashboard.yaml

<br/>

### Kubernetes Dashboard용 Ingress 배포하기

위에서 배포한 Kubernetes Dashboard로 라우팅 되도록 Ingress를 배포한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: "nginx" 
spec:
  rules:
  - host: k8s-dashboard.com
    http:  
	  paths:
      - backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 9090		  
        path: /
        pathType: Prefix

```

> kubectl apply -f k8s-dashboard-ingress.yaml  
<br/>

### Kubernetes Dashboard에 EKS 권한 부여하기

기본적으로 Kubernetes Dashboard 사용자의 권한은 제한되어 있다.  
<br/>

그래서 eks-admin이라는 이름으로 Service Account 및 Cluster Role Binding를 생성하고, 이를 사용하여 관리자 권한으로 Dashboard에 연결할 것이다.  
<br/>

아래와 같이 Service Account와 Cluster Role Binding를 생성한다.   
<br/>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
```

> kubectl apply -f eks-admin-service-account.yaml  
<br/>

### Nginx에 ssl 설정하기

Kubernetes Dashboard는 무조건 https 통신이 돼야 하는데 이를 nginx에서 처리할 수도 있다.  
<br/>

그래서 openssl 명령어로 사설 인증서를 생성한 후 Nginx에 ssl 설정을 해보자.  
<br/>

- openssl 명령어로 key 발급하기  

```bash
# private key와 인증서를 발급한다. 아래 명령어를 입력하면 국가부터 회사까지 각종 정보를 입력해야 한다.
$ openssl req -new -newkey rsa:2048 -nodes -keyout k8s-dashboard.key -out k8s-dashboard.csr

# 위에서 발급한 private key와 csr 키를 통해 crt 키를 발급한다.
$ openssl x509 -req -days 365 -in k8s-dashboard.csr -signkey k8s-dashboard.key -out k8s-dashboard.crt

```
<br/>

- Nginx에 ssl 설정하기  

```bash
upstream k8s-dashboard {
  least_conn;
  # Ingress Controller Service IPs
  server 10.123.45.678:80;
  server 10.123.45.679:80;
}
server {
        # ssl on 대신 listen 구문 마지막에 ssl을 넣는 것으로 변경됨.
        listen 443 ssl;
        server_name k8s-dashboard.com;
        charset utf-8;
		
        access_log /etc/nginx/log/access.log;
        error_log /etc/nginx/log/error.log;
        
		#ssl 인증서 적용
        ssl_certificate /etc/nginx/ssl/k8s-dashboard.crt;        #생성된 인증서경로
        ssl_certificate_key /etc/nginx/ssl/k8s-dashboard.key;    #생성된 개인키
        
		location / {
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Scheme $scheme;
				
                proxy_pass http://k8s-dashboard;
        }
}
```

- Host 설정하기  
<br/>

nginx에 ssl 설정까지 마쳤으면 이제 hosts 파일에 설정한 Kubernetes Dashboard 도메인과 Nginx 서버 IP를 매핑해줘야 한다.  
<br/>

```bash
$ vi /etc/hosts

# 웹서버 IP      # Kubernetes Dashboard 도메인
10.111.22.333  k8s-dashboard.com
```

### Kubernetes Dashboard에 접속하기

주의사항: 반드시 https로 접속해야 한다.  
<br/>

위에서 설정한 도메인으로 접속하면 아래와 같은 UI가 뜬다.  
> https://k8s-dashboard.com  
<br/>

![k8s](https://user-images.githubusercontent.com/67899732/209635254-b6a71ca7-6c97-49fc-8928-7624d078984f.png)

로그인은 위에서 생성한 eks-admin token 값으로 로그인을 할 것이다.  
<br/>

```bash

# Token 확인 명령어
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')

# 출력 예시
Name:         eks-admin-token-b5zv4
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=eks-admin
              kubernetes.io/service-account.uid=bcfe66ac-39be-11e8-97e8-026dce96b6e8

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      <token 값>
```

- 위에서 출력된 token 값을 복사하여 로그인 한다.  
<br/>

![k8s2](https://user-images.githubusercontent.com/67899732/209636495-c85dadaa-68dd-48cb-9690-0c7ec7efec06.png)

- 로그인 하면 위와 같은 화면이 뜬다. 여기서 원하는 리소스를 골라 확인할 수 있다.  
<br/>

끝.

참고 
- [https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/dashboard-tutorial.html] (https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/dashboard-tutorial.html)
- [https://csupreme19.github.io/devops/kubernetes/2021/03/04/kubernetes-dashboard.htm] (https://csupreme19.github.io/devops/kubernetes/2021/03/04/kubernetes-dashboard.html)
