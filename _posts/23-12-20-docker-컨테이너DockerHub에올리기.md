---
layout: post
title:  "[Docker] 컨테이너를 Docker Hub에 올리기"
excerpt: "현재 로컬에서 사용중인 컨테이너를 그대로 Docker Hub에 올려보자"
author: fhdufhdu
date: '2023-12-20 12:39'
categories: [ docker ]
---
Docker 컨테이너를 그대로 이미지로 만들어서 Docker Hub에 올리고 싶다면 아래와 같이 진행하자.

## Docker Hub 로그인하기

[Docker Hub](https://hub.docker.com/)에 들어가서 로그인을 먼저 한다. 회원가입을 진행하지 않았다면 회원가입을 먼저 하자.

개인 Docker Hub Repository에 이미지를 올리기 때문에 로그인은 필수다.

회원가입을 하고 나면 플랜을 선택하라는 화면이 나올 수 있는데, 이때 Personal 플랜으로 진행하면 된다.(무료)

## Repository 만들기

로그인을 하고나서 메인화면에서

![image](https://github.com/fhdufhdu/fhdufhdu.github.io/assets/32770312/b241f086-f56e-4a0b-bdcb-7ba0a4d6d4b5)

위 이미지와 같이 Repository -> Create Repository를 눌러준다.

정보 입력 후 Repostiory를 생성하게 되면 자신의 Repository 화면으로 들어갈 수 있게 된다.

## 컨테이너 commit 하기

현재 자신의 컴퓨터에 존재하는 컨테이너를 image화 하는 과정이다.

``` bash
$ docker commit 자신의컨테이너이름 repository경로:tag

# 실제 입력 예
# tag(=version)는 본인 전략대로 사용하기
$ docker commit es fhdufhdu/test_elasticsearch:1.0
```

## 생성된 이미지를 업로드하기

``` bash
# 로그인, username에 email은 입력하지 말기
$ docker login

#이미지 이름에 있는 repository로 이미지가 업로드되게 된다.
$ docker push fhdufhdu/mim_elasticsearch:1.0
The push refers to repository [docker.io/fhdufhdu/mim_elasticsearch]
ec8582fc54e5: Pushed 
bca54c335b28: Mounted from library/elasticsearch 
de080e772c1a: Mounted from library/elasticsearch 
cfc1ead7498a: Mounted from library/elasticsearch 
245d80f411fb: Mounted from library/elasticsearch 
489e330647ba: Mounted from library/elasticsearch 
cff7c30e604c: Mounted from library/elasticsearch 
3a50035e8665: Mounted from library/elasticsearch 
7074040eb2b7: Mounted from library/elasticsearch 
af7ed92504ae: Mounted from library/elasticsearch 
1.0: digest: sha256:018de9cde96f6304cb8967a5747fe123244af76ef502fd7c5e446dd91b6aa765 size: 2417
```

위 명령어를 실행하면, 이미지를 Repository에 올릴 것이다.

이후 이미지를 받아와서 컨테이너에 올리는 작업은 기존 방식과 동일하게 진행하면 된다.