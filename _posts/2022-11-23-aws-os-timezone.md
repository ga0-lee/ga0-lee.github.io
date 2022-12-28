---
title: AWS Opensearch Timezone 설정하기
author: garit
date: 2022-11-23 10:00:00 +0900
categories: [AWS, Opensearch]
tags: [aws, opensearch]
render_with_liquid: false
---

## AWS Opensearch Timezone 설정하기

### 상황

AWS Opensearch를 사용하여 EKS 로그를 수집하고 Opensearch Dashboard를 통해 로그를 모니터링 한다.  
<br/>

그러나 Opensearch Dashboard에서 로그를 확인하면 @timestamp가 Table 형식에서는 KST로 출력되고, JSON 형식에서는 UTC로 출력이 되는 문제를 발견했다.  
<br/>

JSON 형식으로 데이터를 뽑아서 써야 하기 때문에 다음과 같은 방법으로 Timezone을 KST로 맞추었다.  
<br/>


### Opensearch Dashboard의 Timezone 적용 방식 

먼저, Opensearch Dashboard에서 JSON 형식으로 보는 로그 데이터와 Devtool 또는 Rest API호출을 통해 보는 로그 데이터는 Opensearch에 저장된 원본 데이터다.  
<br/>

그래서 Opensearch에 데이터가 저장될 때 UTC로 저장되면 UTC로 출력되고, KST로 저장되면 KST로 출력된다.  
<br/>

그러나 Opensearch Dashboard의 Table 형식으로 보는 로그 데이터는 Opensearch Dashboard의 Stack Management/Advanced Settings/Timezone for date formatting에서 설정한 Timezone 옵션값에 따라 원본 데이터가 바뀌어 출력된다.  
<br/>

즉, Opensearch에 인덱싱이 될 때 UTC로 저장된 후 Stack Management/Advanced Settings/Timezone for date formatting에서 KST로 설정하면 Table 형식에서만 원본 데이터인 UTC에 +9를 한 KST로 보인다는 것이다.  
<br/>

또한 Opensearch에 인덱싱이 될 때 KST로 저장된 후 Stack Management/Advanced Settings/Timezone for date formatting에서도 KST로 설정하면 Table 형식으로 로그를 찾으려면 KST 기준시간 +9를 한 시간대에서 로그를 검색해야 한다.
> #예시
> Opensearch Indexing -> KST 09:00:00
> -> Opensearch Dashboard에서 Timezone을 KST로 설정 
> -> Opensearch에 저장된 데이터에 Timezone 세팅까지 이중으로 하는 셈이다.
> -> 그래서 KST 18:00:00에서 찾아야 해당 로그를 확인할 수 있다.  

이런식으로 동작되기 때문에 Opensearch Timezone 설정을 하려면 몇 가지 선택을 해야 한다.


### Opensearch 로그의 timestamp를 KST로 출력하기

**1.원본 데이터는 UTC로 저장하고 Opensearch Dashboard에서 Timezone을 설정하여 Table 형식에서만 KST로 로그 보기**  

> opensearch에 인덱싱 하는 lambda 코드 중 아래의 코드는 @timestamp를 UTC로 저장한다.
> source['@timestamp'] = new Date(1 * logEvent.timestamp).toISOString();

위와 같이 Date의 toISOString은 UTC가 기본이라고 한다. 그러므로 위와 같이 UTC로 인덱싱한 후 Opensearch Dashboard에서만 Timezone을 KST로 변경해준다.  
<br/>

방법은 Opensearch Dashboard에 들어가서 좌측 상단의 메뉴를 누르고 Stack Management/Advanced Settings/Timezone for date formatting 경로로 들어가 KST로 변경 후 적용한다.
<br/>

그럼 Opensearch Dashboard Discover에서 로그를 검색할 때 KST 기준으로 검색하면 Table 형식의 로그에서는 timestamp가 KST로 출력된다.  
<br/>

다만 이런 경우에는 JSON 형식으로 확인하면 UTC로 출력된다.  
<br/>

**2.원본 데이터를 KST로 저장하고 Opensearch Dashboard는 UTC로 설정하기**  

> opensearch에 인덱싱 하는 lambda 코드 중 아래의 코드는 @timestamp를 KST로 저장한다.
> source['@timestamp'] = moment(logEvent.timestamp).tz('Asia/Seoul').format('YYYY-MM-DDTHH:mm:ss.SSS[Z]');
- moment 라이브러리를 추가하여 쓰는 방법은 여기서 확인할 수 있다. ->

위와 같이 moment 함수를 사용하여 opensearch 자체에 인덱싱을 KST로 하면 JSON이나 Devtool에서도 KST로 출력되는 로그를 볼 수 있다.  

그러나 이때 몇 가지 더 설정을 해주어야 한다.

- Stack Management/Advanced Settings/Timezone for date formatting 경로로 들어가 UTC로 변경 후 적용한다.  
-> KST에 +9시간이 되는 것을 막기 위해  
<br/>

- Opensearch Dashboard에서 Discover로 검색할 때 now가 아닌 절대 시간을 입력해야 해당 시간의 로그가 검색된다.  
->  아마 Opensearch Dashboard는 UTC로 돼있고 로그 값은 KST라서 현재 시간으로 찾으면 안 나오고 시간 검색에 절대값을 넣어야 검색이 가능한 것 같다.  
<br/>

매번 절대값을 넣어 검색하는 게 번거롭긴 하지만 어쩔 수 없는 것 같다.  
<br/>

차라리 UTC로 데이터를 뽑아서 데이터를 사용할 때 +9시간을 해주는 게 더 나을 수도 있을 것 같다.^^  
<br/><br/>
 


참고 
- [https://aws.amazon.com/ko/blogs/big-data/set-advanced-settings-with-the-amazon-opensearch-service-dashboards-api/] (https://aws.amazon.com/ko/blogs/big-data/set-advanced-settings-with-the-amazon-opensearch-service-dashboards-api/)

