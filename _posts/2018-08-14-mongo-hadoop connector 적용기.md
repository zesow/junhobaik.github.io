## 개요

 현재 유저 별 댓글의 유사도를 비교하는 작업을 하고 있다. 그런데, 댓글의 수가 많다보니 파이썬에서 싱글로 다 돌리기는 불가능했다. 그래서 하둡 mapreduce 를 이용해서 유사도 분석, 즉 댓글 별 TF-IDF 를 계산하려고 한다. 데이터는 mongodb 서버에 들어 있어서, 찾아보니 하둡에서 hdfs를 이용하지 않고 mongo-hadoop connector 를 사용하면 mongo에서 곧바로 가져올 수 있다고 한다. 



## 설치

[공식 repo](https://github.com/mongodb/mongo-hadoop)

WIKI 를 꽤나 잘 해 놔서 많은 도움이 되었다.

우선 다운받아야 하는데, 이 때는 maven을 쓰지 않았으므로 수동으로 다운받아서 보내주었다.

maven을 쓰면 한번에 다운까지는 가능하다.



### git clone & build

위 레포지토리를 clone해서 가져온다. 그 다음 ./gradlew jar 명령어를 이용해 jar 파일들을 build한다. 이러면 우리가 필요로 하는 jar 들이 생성된다.

![](https://www.dropbox.com/s/zu536j54yqiplv8/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-08-14%2017.02.11.jpg?raw=1)



### jar 모으기

우리는 여기서 3가지의 jar 이  필요하다.

1. mongo-hadoop-2.0.2.jar : build/libs 폴더에 있다.
2. mongo-hadoop-streaming-2.0.2.jar : /streaming/build/libs 에 있다.
3. mongo-hadoop-core-2.0.2.jar : /core/build/libs 에 있다.



여기에 추가로, [mongo-java driver](https://mongodb.github.io/mongo-java-driver/) 가 필요하다. 링크로 가서 다운받아주자. 현재 기준으론 3.8 버전이다.

이렇게 4가지를 하나로 묶어서, 우리의 hadoop cluster 각각에 다 보내주면 된다.

`$HADOOP_PREFIX/lib/` 나는 이 폴더에 다 넣어주었다. 공식 문서에는 하둡 버전마다 다르다고 한다.



그런데 이 4가지가 maven dependency 를 이용하면 한 번에 다운로드 된다.

docker image 파일을 만들 때는 이걸로 해야 될 것 같다.



## 테스트

공식으로 제공되는 treasury yield 예제를 실행해 보았다.

위 예제는 mongodb에 있는 데이터를 hadoop mr로 분석한 다음 그 결과를 다시 mongo에 넣는 예제이다.

실행은 간단하다. clone 해 놓은 커넥터에서,

`./gradlew historicalYield`

를 입력해주면 된다. 잘 세팅되어 있다면 문제없이 작동 될 것이다.

![](https://www.dropbox.com/s/wqntudqxsgp0som/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-08-04%2017.26.56.jpg?raw=1)

![](https://www.dropbox.com/s/df5rtkl531jwuf8/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-08-04%2023.06.26.jpg?raw=1)

분석이 완료되는 것을 확인하였다.



## 결론

 무척이나 뻘짓을 많이 한 것에 비해서 정리하니까 매우 간단해 보인다. 

gradle,maven 에 익숙하지 않고 hadoop 및 docker 사용도 아직 미숙해 많은 시간이 걸렸다.

다음에는 직접 우리 데이터에 tf-idf를 구하는 포스팅을 해보겠다.