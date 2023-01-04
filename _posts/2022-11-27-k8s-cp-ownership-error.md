---
title: kubectl cp 명령어 ownership 에러 해결하기
author: garit
date: 2022-11-27 10:00:00 +0900
categories: [Nginx, Cases]
tags: [nginx]
render_with_liquid: false
---

## kubectl cp 명령어 ownership 에러 해결하기

### 상황

kubectl cp 명령어로 bastion host에 있는 파일을 pod에 마운트 되어있는 file system 디렉토리로 복사할 때 다음과 같은 에러가 발생했다.   
<br/>

- 에러 내용
> tar: 파일명: Cannot change ownership to uid xxxx, gid xxxx: Operation not permitted
> tar: Exiting with failure status due to previous errors
> command terminated with exit code 2
<br/>
 

### 해결 방법

위 에러는 bastion host에 있던 파일을 file system으로 옮기면서 file system의 소유자로 권한을 바꿔줘야 하는데 실패해서 발생하는 에러였다.  
<br/>

kubectl cp 명령어를 사용할 때 파일의 소유자를 그대로 유지하는 것이 기본 값이다.  
<br/>

그러므로 kubectl cp 명령어를 사용할 때 --no-preserve=true 파라미터를 넣어주어 파일의 소유자를 유지하지 않게 하면 된다.   
<br/>

> 기본 값은 --no-preserve=false 
<br/>
