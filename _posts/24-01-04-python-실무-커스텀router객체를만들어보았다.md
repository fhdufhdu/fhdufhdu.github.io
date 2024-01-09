---
layout: post
title:  "[Python] [실무] 커스텀 Router 객체를 만들어 보았다."
excerpt: "직접 만들어보는 Router 객체! 이를 통해 속도 개선 경험을 공유한다"
author: fhdufhdu
categories: [ python ]
date: '2024-01-04 23:10'
---
## 문제
​
회사에서 필자가 개발한, 멀티 챗봇 시스템에 VOC가 들어오게 되었다. 해당 내용은 "가끔씩 너무 챗봇이 느려요"였다.
​
챗봇 모델 쪽에서 응답이 늦는 경우는 자주 있었기에, 이번에도 그렇겠거니 하고 확인을 해보았다.
​
그런데 로그에 찍힌 시간을 보니, 모델은 빠른 시간 내에 응답을 해주는 것을 확인했다. 거기서 문제가 있다는 것을 깨닫고, 문제점을 찾아가기 시작했다.
​
문제는 importlib이라는 python 기본 라이브러리가 문제였다. 해당 라이브러리를 사용할 때 간헐적으로 최대 1.8초나 지연되는 현상이 발생했다.
​
### 기존 라우팅 방식
​
우선 간략하게 현재 개발 환경을 말하자면 아래와 같다.
​
-   AWS Lambda(람다)
-   AWS Api Gateway
-   PostgreSQL
-   Python
-   웹 프레임워크 사용하지 않음
​

챗봇 서버는 웹 프레임워크를 사용하지 않았기에, 챗봇별로 로직을 실행할 때 `importlib`이라는 라이브러리를 이용해서 라우팅을 실시했다. `importlib`은 동적으로 import를 할 수 있게 도와주는 라이브러리이다.
​
예를 들어 body에 `chatbotId`가 `Foo`이면 `FooFacade` 클래스를 import 해서 쓰거나, `Bar`이면 `BarFacade` 클래스를 import해서 쓰거나 하는 식으로 진행했다.
​
그런데 `importlib`이 느리다니... 이해가 되지 않았다. `importlib`은 결국 `__import__` 함수의 래퍼이다. `__import__`함수는 import 구문을 만나면 실행되는 기본적인 함수이다. 이게 느리다면, 파이썬을 사용해도 되는 것일까?
​
### Import의 작동방식에 대해서 알아보자
​
해당 정보는 [김지연 님의 블로그](https://medium.com/@likegondry/python-til-%ED%8C%8C%EC%9D%B4%EC%8D%AC%EC%9D%B4-import%EB%A1%9C-%ED%8C%8C%EC%9D%B4%EC%8D%AC%EC%9D%84-%EB%B6%88%EB%9F%AC%EC%98%A4%EB%8A%94-%EB%B0%A9%EB%B2%95-76e268e7613b)를 참고해서 작성했다.
​
#### 1\. sys.module에 모듈이 존재하는지 찾아보기
​
sys.module에는 이때까지 사용했던 module들이 딕셔너리 형태로 저장되어 있다. import시 해당 모듈이 이전에 import 된 것이라면 빠르게 가져올 수 있다.
​
#### 2\. sys.path에 저장된 파일 목록들 하나하나 찾아보기
​
이 작업이 좀 오래 걸린다. 파일 리스트을 하나하나 탐색하면서 모듈을 가져오기 때문에 시간이 오래걸린다. 아마 필자의 생각으로는 File I/O 작업이라서 오래 걸리는 것 같다.
​
#### 그럼 동적으로 import 하는 것은...?
​
만약 `FooFacade` 클래스를 **처음** 동적으로 import 한다면, 생각보다 시간이 오래 걸릴 수 있겠다는 생각이 들었다. 실제로도 처음 실행할 때와, 조금 유휴시간이 지난 후 실행하면 `importlib` 동작 시간이 오래 걸리는 것을 확인할 수 있었다.
​
심지어 AWS 람다를 이용하고 있어서, 일정 유휴시간이 지나면 컨테이너가 내려가버린다. 그렇다면 새롭게 컨테이너가 생성될 때마다, `importlib`에서 시간을 많이 잡아먹었다.
​
## 해결방법
​
이제 문제점을 찾았으니 해결을 해보자.
​
결국 라우팅의 문제였으니, 이 라우팅을 다른 방식으로 하면 되지 않을까?
​
그래서 유명한 파이썬 웹 프레임워크인 FastAPI의 [깃허브 소스](https://github.com/tiangolo/fastapi)를 뜯어보았다.
​
FastAPI는 어떻게 라우팅을 사용하고 있을까?
​
```
from fastapi import APIRouter, FastAPI
​
app = FastAPI()
internal_router = APIRouter()
users_router = APIRouter()
​
@users_router.get("/users/")
def read_users():
    return [{"name": "Rick"}, {"name": "Morty"}]
​
internal_router.include_router(users_router)
app.include_router(internal_router)
```
​
이런 식으로 `APIRouter` 객체를 하나 생성하고, APIRouter 객체의 `get`(`post`, `put`, ...) 함수를 라우팅 하고자 하는 함수에 데코레이터로 붙여준다. 그리고 `app` 객체에 해당 라우터를 전달한다.
​
이후 http 요청이 들어오면 `app` 객체로 전달되고, `app` 객체는 라우팅 정보를 확인해서 해당 함수를 실행한다.
​
자 어떻게 이것이 가능할까.
​
필자의 생각은 아래와 같았다.
​
1.  `APIRouter` 객체는 멤버 함수로 데코레이터로 사용가능한 함수를 가지고 있다.(ex. `get`, `post`, `put`, `patch`, `delete`)
2.  데코레이터 함수는 자기가 붙은 함수를 객체 형태로 사용할 수 있다.  
    (ex. `user_router`의 get 함수는 `read_users` 함수를 객체형태로 사용가능)
3.  그렇다면 get을 호출하면, `read_users` 같은 함수를 `users_router`에 딕셔너리 형태로 저장하면 되겠네?  
    (ex. `{"users": read_user}` 와 같은 형태로)
4.  맞는 것 같은데... 한번 확인해 볼까?
​
`APIRouter`의 `get` 함수를 보면 `self.api_route(...)`를 호출하고 해당 결괏값을 바로 반환한다.
​
그렇다면 `api_route` 함수를 보자.
​
``` python
# 너무 길어서 간략하게 축소한 버전이다.
    def api_route(
        self,
        path: str,
        **kwargs,
    ) -> Callable[[DecoratedCallable], DecoratedCallable]:
        def decorator(func: DecoratedCallable) -> DecoratedCallable:
            self.add_api_route(
                path,
                func,
                **kwargs
            )
            return func
​
        return decorator
```
​
해당 함수를 보게 되면, `user_router.get`이 `read_users`에 데코레이터로 붙게 되는 순간 `func` 파라미터에 `read_users가` 들어오게 된다. 이후 `self.add_api_route`를 호출하는데, 이때 아래와 같은 코드가 실행된다.
​
``` python
    def add_api_route(
        self,
        path: str,
        endpoint: Callable[..., Any],
        **kwargs,
    ) -> None:
        """
        생략
        """
        route = route_class(
            self.prefix + path,
            endpoint=endpoint,
            """
            생략
            """
        )
        self.routes.append(route)
```
​
path는 "/user/", endpoint는 `read_users` 함수이다. 이 두 가지를 통해 `route` 객체를 하나 만들고 이를 `user_routes의` `routes` 리스트에 추가한다.
​
필자가 생각한 3번 과정은 아니고, 리스트 탐색으로 라우팅을 하는 것이지만, 어찌 됐든 비슷하다고 생각했다.
​
그리고 path param을 생각하면 딕셔너리의 key, value 탐색보다 리스트 탐색이 더 낫다고 생각이 든다.(path param이 들어가면 어쨌든 n만큼 순회해야 하니까!)  
  
​
필자의 생각이 어느 정도 맞다는 걸 인지했으니 이제 개발해서 라우팅을 적용시키는 일만 남았다.
​
## 커스텀 Router 클래스
​
-   Router
​
``` python
# router.py
from typing import Any, Dict, List
​
class Router:
    def __init__(self):
        self._data: Dict[str, Any] = {}
    
    def _make_key(self, key_list:List[str]) -> str:
        return "-".join(key_list)
    
    def add(self, key_list:List[str]):
        def decorated(func):
            self._data.setdefault(self._make_key(key_list), func)
            return func
        return decorated
       
    def include_router(self, router):
        if type(router) != Router:
            raise Exception("Router 객체만 include 할 수 있음")
        self._data.update(router._data)
    
    # cls_constructuere_args/cls_constructure_kwargs 는 불러올 클래스의 생성자에 필요한 파라미터
    def get_instance(self, key_list:List[str], *cls_constructure_args, **cls_constructure_kwargs):
        _cls = self._data.get(self._make_key(key_list), None)
        return _cls(*cls_constructure_args, **cls_constructure_kwargs) if _cls else None
```
​
-   사용 예제
​
``` python
# foo_facade.py
from router import Router
​
foo_router = Router()
​
@foo_router.add(["foo", "facade"])
class FooFacade:
    def __init__(self, call_str: str):
        self.call_str = call_str
        
    def call(self):
        print(f"Call FooFacade ({self.call_str})")
```
​
``` python
# main.py
from foo_facade import foo_router, FooFacade
from router import Router
​
main_router = Router()
​
main_router.include_router(foo_router)
​
foo_facade:FooFacade = main_router.get_instance(["foo", "facade"], "hi foo facade")
foo_facade.call()
​
# Call FooFacade (hi foo facade) 가 출력됨
```
​
-   main.py 실행 결과
​
![image](https://github.com/fhdufhdu/fhdufhdu.github.io/assets/32770312/5257aff9-29b3-4371-b2c1-ec2beb422ac7)
​
## 결과
​
무려 라우팅시 1.8초나 걸리던 것이 0.001초 미만에 해결되는 모습을 보였다. 해당 코드를 만들고 동료 개발자와 코드리뷰 때 "어떻게 이런 생각을 했냐"라고 하셔서 되게 기분이 좋았다.