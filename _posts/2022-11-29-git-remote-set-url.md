---
title: 기존 git 소스를 새로운 git repo로 복사하기
author: garit
date: 2022-11-29 10:00:00 +0900
categories: [Git, Cases]
tags: [git]
render_with_liquid: false
---

## 기존 git 소스를 새로운 git repo로 복사하기

### 상황

기존에 쓰던 git 소스를 새로운 repo로 옮겨야 하는 상황이 생겼다.    
<br/>

그래서 git remote 명령어를 통해 옮겨보고자 한다.  
<br/>

### 해결 방법

아래와 같이 git 명령어를 통해 a-repo에 있던 모든 소스를 b-repo로 옮긴다.    
<br/>

```bash
# a-repo를 clone한 dir로 이동
cd a-repo

# 원격 저장소 목록을 확인하는 명령어
git remote
-> 예시 출력) origin

# origin이라는 원격 저장소의 url을 가져오는 명령어
git remote get-url origin
-> 예시 출력) https://git.com/a-repo.git

# origin이라는 원격 저장소의 url을 새로운 url로 변경하는 명령어
# git remote set-url [원격저장소명] [new-repo-url]
git remote set-url orgin https://git.com/b-repo.git

# 변경됐는지 확인하는 명령어
git remote get-url origin
-> 예시 출력) https://git.com/b-repo.git


```
