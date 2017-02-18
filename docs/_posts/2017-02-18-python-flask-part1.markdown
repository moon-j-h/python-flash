---
layout: post
title:  "part1"
date:   2017-02-18 18:47:50 +0900
categories: translation
---

이 포스터는 [원문](https://realpython.com/blog/python/flask-by-example-part-1-project-setup/)을 의역하고 필요한 부분은 추가한 것.


> Part one : 로컬 개발 환경을 설치하고 Heroku의 staging과 production 환경에 양 쪽에 배포하자

# Project Setup

Heroku의 staging과 production환경에 있는 "Hello World" 앱으로 시작해보자.

초기 세팅을 위해, 아래의 tool들을 참고하라

 - Virtualenv - [http://www.virtualenv.org/en/latest/](http://www.virtualenv.org/en/latest/)
 - Flask - [http://flask.pocoo.org/](http://flask.pocoo.org/)
 - git/Github - [http://try.github.io/levels/1/challenges/1](http://try.github.io/levels/1/challenges/1)
 - Heroku(basics) - [https://devcenter.heroku.com/articles/getting-started-with-python]( https://devcenter.heroku.com/articles/getting-started-with-python)

먼저, 작업 디렉토리를 설정하자 :

{% highlight %}
$ mkdir flask-by-example
$ cd flask-by-example
{% endhighlight %}

