---
layout: post
title:  "part6"
date:   2017-02-27 17:20:07 +0900
categories: translation
---

이 포스팅은 [원문](https://realpython.com/blog/python/updating-the-staging-environment/)을 의역하고 필요한 부분은 추가한 것.


> Part six : Heroku의 staging sever 에 push하자 - Redis를 설정하고 single Dyno에서 두개의 process(web과 worker)가 어떻게 돌아가는 지 자세하게 알아보자.

# Test Push 

현 code를 staging에 push하고 어떤 것을 준비해야하는지 확인하라.

```python
$ cd flask-by-example
$ git add -A 
$ git commit -m "added angular and the backend worker process"
$ git push stage master
$ heroku open --app wordcount-stage
```

> `wordcount-stage`를 너의 app의 이름으로 바꾸는 것을 잊지말자

word counting 이 잘 되는 지 빠르게 시도해 보자. 아무것도 일어나지 않는다. 왜 그런가?
"Chrome Developer Tools"의 "Network" tab을 열어보면 500(Internal servel error) state code로 return된 `/start`의 POST request가 보일 것이다.

local에서 어떻게 실행시킬 것인지 생각해보자. worker process와 Flask Development Server와 함께 Redis server를 실행시켜라.
같은 일에 Heroku에서도 일어난다.

# Redis

staging app에 Redis를 추가하자:
```python
$ heroku addons:create redistogo:nano --app wordcount-stage
```
다음의 command로 환경변수로 `REDISTOGO_URL` 확인하라.
```python
$ heroku config --app wordcount-stage | grep REDISTOGO_URL
``` 

이미 설정한 코드에서 Redist URL를 연결했는지 확인해야한다.
*worker.py*을 열어서 다음의 코드를 확인하라:
```python
redis_url = os.getenv("REDISTOGO_URL", "redis://localhost:6379")
```

먼저 `REDISTOGO_URL`이라는 환경변수를 참조하여 해당 URI로 테스트 할 것이다. 만약 환경변수가 없다면,
`redis:localhost:6379` URI를 사용한다.

>  Redis와 어떻게 작동하는지 더 확인하고 싶다면 [공식 heroku documents](https://devcenter.heroku.com/articles/redistogo)를 확인하라.

Redis를 설정했으면, 이제 worker process를 실행하자.

# Worker

Heroku는 하나의 dyno를 무료로 사용할 수 있게 한다. 그리고 한 개의 dyno 당 한개의 process를 사용할 수 있다.
우리의 경우에는, web process가 하나의 dyno에 돌고 , worker process가 다른 것에서 돌아야한다.
그러나, 작은 project 이기 때문에, 차선택으로 두 개의 process를 하나의 dyno에 실행하도록 배포할 수 있다.
하지만 이 방법은  큰 project에는 추천하지 않는 방법임을 기억해라.

제일 먼저, root directory에 *heroku.sh*라는 bash script를 추가하라.
```python
#!/bin/bash
gunicorn app:app --daemon
python worker.py
```

그 다음, *Procfile*을 업데이트하자:
```python
web: sh heroku.sh
```
이제 web(demonized, in the background) 과 worker(in the foreground) process 둘다 *Procfile*에서 web process 아래에서 실행된다.

> Heroku에서 무료로 web과 worker process 둘다 실행하는 다른 방법을 찾아봐라. 다른 post에 대안을 보여줄 것이다.

staging server에 push 하기 전에 local에서 test해자.
새로운 terminal 창에서 Redis server를 실행하자 - `redis-server`
그 다음 [heroku local](https://devcenter.heroku.com/articles/heroku-local)을 실행하자. 

```python
$ heroku local
forego | starting web.1 on port 5000
web.1  | 18:17:00 RQ worker 'rq:worker:Michaels-MacBook-Air.9044' started, version 0.5.6
web.1  | 18:17:00
```

[http://localhost:5000/](http://localhost:5000/)로 이동하고 application을 test해보자. 작동할 것이다.
변경된 것을 commit하고 Heroku에 push하라.


[원문 : flask-by-example-part6](https://realpython.com/blog/python/updating-the-staging-environment/)