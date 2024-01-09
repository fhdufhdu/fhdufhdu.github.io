---
layout: post
title:  "[PostgreSQL] [실무] Select 쿼리 속도를 개선해보자 1편"
excerpt: "ORM으로 잘못 짜여진 쿼리를 수정해보자"
author: fhdufhdu
categories: [ postgresql ]
date: '2024-01-07 20:35'
---
##  문제

챗봇들의 게시글을 조회하는데 15초가 걸려버리는 일이 발생했다. 해당 이슈를 듣자마자 바로 SQL 쿼리 문제라는 생각이 들었다. 역시나 예상대로 쿼리가 문제였다.

정확히 말하면, ORM이 문제였다. 해당 백엔드 서버는 초기 개발을 외주업체에서 진행했는데, 듣자 하니 되게 시간이 촉박했다고 한다. 그래서 그런지 모든 데이터베이스 접근이 ORM으로만 되어 있었다.

그래서 서버 코드를 인수인계받고 필자가 제일 먼저 했던 작업이 raw SQL도 사용할 수 있게끔 하는 것이었다.

이번 문제의 쿼리도 ORM으로 작성된 쿼리였다.

이제 한번 문제를 살펴보자

### 테이블 구조

### ORM 코드와 원인

``` js
const contents = await this.contentRepository.createQueryBuilder("c")
	.select(["c.id", "c.content", "b.id", "b.name"])
	.leftJoinAndSelect("c.bot", "b")
	.leftJoinAndSelect("c.bookmark", "b2", "b2.userId = :userId", {userId:1})
	.leftJoinAndSelect("c.like", "l")
	.addSelect((qb) => {
		return qb.subQuery()
			.select("count(cc.id)")
			.from(Comment, "cc")
			.where("cc.contentId = c.id")
	}, "commentCount")
	.skip(0)
	.take(20)
```

이렇게 코드가 작성되어 있었다. 해당 코드를 SQL로 바꿔보면 아래와 같다.

``` sql
select c.id, c,content, b.id, b,name, 
	(select count(*) from "Comment" cc where cc.content_id = c.id) as "commentCount"
from "Content" c
	  left join "Bot" b
		  on c.bot_id = b.id
	  left join "Bookmark" b2
		  on b2.content_id = c.id on b2.user_id = 1
	  left join "Like" l
		  on l.content_id = c.id
offset 0
limit 20;
```

이 쿼리의 문제점이 뭘까?

우선 위의 쿼리에서 서브쿼리를 뺀 결과가 아래와 같다고 하자.

| c.id | c.content | b2.id | l.id |
| --- | --- | --- | --- |
| 1 | "반가워요" | 1 | 1 |
| 1 | "반가워요" | 1 | 2 |
| 1 | "반가워요" | 1 | 3 |
| 2 | "안녕" | 4 | 5 |
| 2 | "안녕" | 4 | 6 |

원하는 결과는 분명 c의 리스트이다. 하지만 결과를 보면 c가 중복되는 것들이 많이 보일 것이다.

그 중복되는 모든 행에 전부 서브쿼리가 실행되어서 문제가 되는 것이다. 20개의 게시글을 요청했는데, 결과 row은 100개 혹은 200개가 넘어가는 상황이 발생하고 100,200개의 행에 전부 서브쿼리가 실행되니까 계속해서 느려지는 것이었다.

서브쿼리 자체는 크게 문제가 없었다.(사실 join 맺어서 group by로 count 계산하는게 훨씬 빠르다. 이건 2편에서 다루도록 하겠다.) 

## 원인파악 완료! 개선해보자

#### Row를 줄이자.

Like 테이블을 join 한 이유는 유저가 해당 게시글에 좋아요를 눌렀나를 보기 위한 장치이다. 즉, 위의 결과처럼 l.id가 여러 개가 나올 필요가 없다. join을 맺었을 때, 유저가 좋아요를 눌렀다면 l.id가 존재할 것이고, 그렇지 않다면 l.id가 null이 되도록 수정하면 된다.

그렇다면, 한 가지만 수정하면 될 것처럼 보이지 않는가?

``` js
const contents = await this.contentRepository.createQueryBuilder("c")
	.select(["c.id", "c.content", "b.id", "b.name"])
	.leftJoinAndSelect("c.bot", "b")
	.leftJoinAndSelect("c.bookmark", "b2", "b2.userId = :userId", {userId: 1})
	.leftJoinAndSelect("c.like", "l", "l.userId = :userId", {userId: 1})
	.addSelect((qb) => {
		return qb.subQuery()
			.select("count(cc.id)")
			.from(Comment, "cc")
			.where("cc.contentId = c.id")
	}, "commentCount")
	.skip(0)
	.take(20)
```

차이점은 `.leftJoinAndSelect("c.like", "l")`이 `.leftJoinAndSelect("c.like", "l", "l.userId = :userId", {userId: 1})` 로 조건이 추가되었다는 것!

이렇게 한다면 위에서 말한 것처럼 작동한다.

기존 코드에서는 해당 유저가 좋아요를 눌렀는지를 코드상에서 `isLike: !!likedUsers.find((like) => like.userId === userId)` 이렇게 판단했지만, 저렇게 수정하고 나면 row 개수도 게시글 조회 개수에 딱 맞게 조회되며 `isLike: !!likeUsers.length` 로 판단할 수 있게 된다.

## 후기

생각보다 간단하게 수정되어서 조금은 허탈했다.

코드의 의도를 읽기 위해서 온갖 시도를 했었는데(실제 코드는 위의 예시보다 훠어어얼씬 복잡하다.) 의도를 파악하고, 문제를 파악하고 나니, 정말 어이없는 곳에서 문제가 있었던 케이스였던 것 같다.

이 트러블슈팅 경험으로 인해 ORM에 대한 단점이 더 눈에 들어오더라. ORM이 마냥 좋은 것만은 아니라는 건 체감하고 있었지만, 이번 경험으로 인해 그 생각이 확고히 굳어진 것 같다. SQL, 그리고 DBMS에 대한 이해 없이 ORM만 사용하게 된다면, 좋은 개발자가 될 수 없을 것이라고 생각한다.

앞으로도 여러 가지 방법을 잘 조합해서 사용해 보도록 해야겠다. 이번에는 ORM 수정으로만 끝났지만, 2편에서는 해당 데이터 조회에서는 이번 포스팅에서 수정한 ORM 대신 SQL을 사용하게 된 경험에 대해 얘기하도록 하겠다.