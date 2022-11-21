---
title: helm 개념 및 기본 사용법
author: garit
date: 2022-11-15 10:00:00 +0900
categories: [Kubernetes, Helm]
tags: [k8s, helm]
render_with_liquid: false
---

## helm 개념 및 기본 사용법

### 용어

- Chart: 패키지 같은 것
- Release: 차트를 배포한 인스턴스 -> 여러 개 가능 
- built in object: 빌트 인 객체 -> 객체를 가지고 yaml 파일의 값을 지정할 수 있다. 
  - ex) {{ .Release.Name }}

### 기본 명령어

```bash
# Chart 생성하기
helm create [chart명]

# Release 이름 지정해서 생성하기
helm install [release명] [chart_dir_path] 
# Release 이름은 자동으로 주고 생성하기
helm install [chart_dir_path] --generate-name
# Release 생성 dry-run
helm install [release명] [chart_dir_path] --dry-run --debug
# Release 생성시 변수로 들어가는 값 직접 지정하기
helm install [release명] [chart_dir_path] --set [key=value]

# Release 목록 출력
helm list

# Release의 manifest 확인하기
helm get manifest [release명]
```



- 참고
    - [https://helm.sh/ko/docs/chart_template_guide/getting_started/](https://helm.sh/ko/docs/chart_template_guide/getting_started/)
    
