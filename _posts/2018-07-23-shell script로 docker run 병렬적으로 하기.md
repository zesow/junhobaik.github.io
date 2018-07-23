---
- tags
  - docker
  - shell script
  - parallel processing
---



## 개요

 저번 포스팅에서 docker run 해서 원하는 페이지를 크롤링하는 이미지를 만들었다.

이번 포스팅은 이 이미지로 여러 개의 container 를 만들어서 각각 페이지를 할당하도록 script를 짜려고 한다.



## 조건

스크립트는 다음과 같은 기능을 수행해야 함.

1. 해당 쇼핑몰의 총 페이지 수를 계산해야 함.
2. 그 페이지 수를 N 개의 container 에게 N등분 하여 분배해야 함.
3. N개의 container를 run 해야 함.



## 진행



### 1. 해당 쇼핑몰의 총 페이지 수를 계산해야 함

저번에 페이지 수를 계산하는 코드를 이미 만들어 놓았다. 

고민은 그 파이썬 코드를 쉘 스크립트에서 어떻게 실행하냐 하는 것이였는데, 간단히 쉘 스크립트에서 파이썬 스크립트를 실행하면 print 값을 받아왔다.

`ret=python3 lastpage_artlance.py`



expr 을 이용하여 사칙연산도 가능하다.

`echo expr $ret / 2`



### 2. 그 페이지 수를 N 개의 container 에게 N등분 하여 분배해야 함.

오늘 기준으로, 쇼핑몰 중 최대 페이지 수는 39033 개였다.

약간 많아지는 것 같지만, 일단 페이지 1000개 당 하나의 컨테이너를 할당하기로 하고 진행하기로 했다.

구문은

`docker run -d -v /tmp/data:/data my_crawler_0.1 시작페이지 끝페이지 &`

로 쉘스크립트의 for 문을 이용해서 반복한다.

Tip) docker 한번에 종료

`sudo docker rm $(docker ps -aq)`



### 3. N개의 container를 run 해야 함.

쉘스크립트를 이용한 코드는 다음과 같다.

```shell
#!/bin/bash

pNum=`python3 lastpage_artlance.py`
sPage=1
ePage=0
echo '총페이지: '$pNum
while [ $sPage -le $pNum ]
do
	ePage=`expr $sPage + 999`

	if [ $ePage -gt $pNum ]
	then
		ePage=$pNum
	fi

	echo '시작페이지:'$sPage', 끝페이지:'$ePage' 의 container 실행'
	`docker run -d --name art_$sPage-$ePage -v /tmp/data:/data my_crawler_0.1 $sPage $ePage`

	sPage=`expr $sPage + 1000`
done
```

쉘 스크립트는 대소 비교부터 표현이 조금 생소했다.

[쉘스크립트 문법 1](http://www.fun-coding.org/linux_basic3.html)



![](https://www.dropbox.com/s/tagwttdc3s0u8iu/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-07-23%2014.45.56.jpg?raw=1)

실행 결과. 성공적으로 페이지 1000개씩 container가 돌아가고 있는 걸 확인할 수 있다.



## 결론

   병렬로 실행되게까지 완료했다.

다음 포스팅에서는 크롤링이 끝난 json 파일을 mongodb에 넣고 기존 table 과 비교하는 기능을 수행하는 스크립트를 작성할 것이다.