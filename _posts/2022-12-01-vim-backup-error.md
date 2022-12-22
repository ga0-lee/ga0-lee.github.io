---
title: vim 에러 Cannot create backup file(add ! to overwrite) 해결하기
author: garit
date: 2022-11-15 10:00:00 +0900
categories: [Linux, Cases]
tags: [linux, vim]
render_with_liquid: false
---

## vim 에러 Cannot create backup file(add ! to overwrite) 해결하기

### 상황
vim 명령어로 파일을 수정하고 :wq로 저장하려 하면 아래와 같은 에러가 발생한다.  
> Cannot create backup file(add ! to overwrite)
<br/>

- 이런 에러는 !를 입력하면 해결할 수 있지만 파일을 수정할 때마다 매번 발생하므로 번거롭기 때문에 해당 에러 메세지가 나오지 않도록 해결해야 한다.  
<br/>

### 해결 방법

.vim 디렉토리에 backups 디렉토리를 생성해준다.

```bash
cd ~/.vim 
mkdir backups
```

이제 더이상 해당 에러 메세지가 출력되지 않고 :wq만으로도 수정이 잘 된다.
<br/><br/>


참고 
- [https://stackoverflow.com/questions/8428210/cannot-create-backup-fileadd-to-overwrite](https://stackoverflow.com/questions/8428210/cannot-create-backup-fileadd-to-overwrite)

