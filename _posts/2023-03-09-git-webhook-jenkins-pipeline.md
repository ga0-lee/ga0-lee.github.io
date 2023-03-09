---
title: Jenkins Pipeline을 Git Webhook으로 빌드하기
author: garit
date: 2023-03-09 15:00:00 +0900
categories: [Git, Cases]
tags: [git]
render_with_liquid: false
---

## Jenkins Pipeline을 Git Webhook으로 빌드하기

### 상황

Jenkins Pipeline을 Git Webhook과 연동하여 git tag가 생성되었을 때 Jenkins에서 빌드가 자동으로 시작되게 설정하고자 한다. 
<br/>

### Jenkins 설정

먼저 User Token을 생성한다.  
<br/>
> #생성 방법  
> Jenkins 관리 -> Manage Users -> User 클릭 (ex.admin) -> 설정 -> API Token -> Add new Token 
> -> Token 이름 입력(ex. webhook) -> Generate 버튼 클릭하여 Token 생성 -> Token 값 저장 -> Save  
<br/>

그리고 Pipeline을 생성한다.  
<br/>

> #설정 방법  
> 새로운 Item -> Job 이름 입력 -> Pipeline 선택 -> Build Triggers에서 빌드를 원격으로 유발(예: 스크립트 사용) 선택 > -> Authentication Token 이름에 위에서 생성한 User Token 이름을 입력 -> Token 입력칸 아래 URL 따로 저장-> Pipeline 작성 -> 저장
<br/> 

- 위에서 따로 저장한 빌드 유발 URL을 다음과 같이 수정한다.
> http://User이름:Token값@Jenkins주소/view/Papyrus-build/job/dev-messenger-front/buildWithParameters?token=Tokens이름  
> 예시) http://admin:1231313213@test.jenkins.co/view/Papyrus-build/job/dev-messenger-front/buildWithParameters?token=webhook
<br/> 

### Github 설정

이제 위에서 생성한 Jenkins Job을 자동으로 실행시킬 Webhook을 설정한다.  
<br/>

> #설정 방법  
> Repository 접속 -> Settings -> Webhooks -> Add webhook -> Payload URL에 위에서 수정한 빌드 유발 URL을 입력한다 (http://User이름:Token값@Jenkins주소/view/Papyrus-build/job/dev-messenger-front/buildWithParameters?token=Tokens이름) -> Content type: application/json -> Which events would you like to trigger this webhook? (어떤 이벤트로 빌드를 실행시킬 건지 선택): Let me select individual events.에서 Branch or tag creation(branch나 tag가 생성될 때) 선택 -> Active는 활성화 -> Add webhook 
<br/>

> #webhook 결과 확인 방법  
> Webhooks 페이지 -> 위에서 생성한 webhook 선택 -> Recent Deliveries 클릭 -> 여기서 webhook의 성공/실패 여부를 확인할 수 있다.  
> 에러가 발생하는 경우 설정을 변경하고 Redeliver를 클릭하여 재시도 해볼 수 있다.
<br/>

### 403 에러 해결법
- 처음 발생한 403에러는 403 Forbidden 에러였는데 이건 Jenkins 방화벽 설정 때문에 발생한 에러였다.  
    내 경우에는 K8s에 Pod로 젠킨스를 사용 중이었고, Nginx Ingress Controller를 사용하여 라우팅 설정을 했다.  
    그래서 아래와 같이 annotation에 Github의 webhook ip range를 whitelist에 추가해주면 된다.  

- Git IP 확인: https://api.github.com/meta
> webhook IP: 192.30.252.0/22,185.199.108.0/22,140.82.112.0/20,143.55.64.0/20

<img width="648" alt="image" src="https://user-images.githubusercontent.com/67899732/223937309-5d49a1c4-2eb0-4348-868e-5006ed06a98e.png">

- 두 번째로 발생한 403에러는 HTTP ERROR 403 No valid crumb was included in the request 였다.  
    이 에러는 Payload URL을 잘못 입력해서 발생했던 에러였다. Token값이나 user이름 등을 잘못 쓰는 경우.  


### Webhook으로 빌드 Trigger 되는지 테스트하기

git tag를 생성하여 webhook을 동작시켜 젠킨스 빌드를 실행한다.  


```bash
# git tag 생성
git tag test-0.1
# 생성한 tag를 repo에 push
git push origin test-0.1
```

- 위 명령어로 tag를 생성하면 webhook이 동작하며 젠킨스 빌드가 실행된다.  

- 아래 화면은 webhook이 tag create로 동작했다는 것을 보여준다.  

<img width="838" alt="image" src="https://user-images.githubusercontent.com/67899732/223940363-d9bd1509-5220-4d4f-833b-4ef50697bb7a.png">

- 그리고 젠킨스에서도 빌드가 실행된 것을 확인할 수 있다.

