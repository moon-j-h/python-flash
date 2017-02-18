---
layout: post
title:  "part1"
date:   2017-02-18 18:47:50 +0900
categories: translation
---

이 포스티은 (원문)[https://realpython.com/blog/python/flask-by-example-part-1-project-setup/]을 의역하고 필요한 부분은 추가한 것.


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

작업 디렉토리에 새로운 git repo를 초기화하자 : 

{% highlight %}
$ git init
{% endhighlight %}

application에 사용할 가상 환경을 설치하자 : 

{% highlight %}
### linux
$ pyvenv-3.5 env

$ source env/bin/activate
{% endhighlight %}

{% highlight %}
### windows
>python -m venv env

>env\Scripts\activate
{% endhighlight %}

너는 지금 가상 환경에서 작업 중이란 것을 알려줄 `(env)` 표시를 terminal의 왼쪽 prompt에서 볼 수 있다.

> ### 가상 환경을 유지하기 위해서는, 그냥 `deactivate`로 실행하고, project를 다시 실행하고 싶을 때 `source env/bin/active`를 해라

다음으로, app을 위한 기본 구조를 설정하자. 그리고 "flask-by-example" 폴더에 다음의 파일을 추가하자.

{% highlight %}
### linux
touch app.py .gitignore README.md requirements.txt
{% endhighlight %}

{% highlight %}
### windows
>copy /b NUL app.py
>copy /b NUL .gitignore
>copy /b NUL README.md
>copy /b NUL requirements.txt
{% endhighlight %}

위 명령어를 실행하면, 아래의 파일들이 생성될 것이다

{% highlight %}
├─── .gitignore
├─── app.py
├─── README.md
└─── requirements.txt
{% endhighlight %}

다음으로, Flask를 설치하자 : 

{% highlight %}
$ pip install Flask==0.10.1
{% endhighlight %}

그리고 *requirements.txt*에 설치된 라이브러리를 추가하자 :

{% highlight %}
$ pip freeze > requirements.txt
{% endhighlight %}

*app.py*을 열어 다음의 코드를 입력하자 : 

{% highlight % python}
from flask import Flask
app = Flask(__name__)


@app.route('/)
def hello():
  return "Hello World!"

if __name__ == '__main__':
  app.run();

{% endhighlight %}

app 을 실행하자 : 

{% highlight %}
$ python app.py
{% endhighlight %}

그러면 []()에서 "Hello World" 앱을 볼수 있다. 확인하고 서버는 죽여라.

이제, production 과 stagin app의 Heroku 환경을 설정하자.

# Heroku Setup

준비가 안됐다면, Heroku에 가입하고, Heroky [Toolbelt](https://devcenter.heroku.com/articles/heroku-cli)를 설치하고,
터미널에 `heroku login` 을 실행하서 Heroku에 로그인해라.

먼저, 너의 project의 root 디렉토리에 [Procfile](https://devcenter.heroku.com/articles/procfile)를 생성하라 :

{% highlight %}
### linux
$ touch Procfile
{% endhighlight %}

{% highlight %}
### windows
>copy /b NUL Procfile
{% endhighlight %}

새롭게 생성된 파일에 다음을 입력하라 : 

{% highlight %}
web: gunicorn app:app
{% endhighlight %}

*requirements.txt* 파일에 [gunicorn](http://gunicorn.org/)을 추가하자 : 
{% highlight %}
$ pip install gunicorn==19.4.5
$ pip freeze > requirements.txt
{% endhighlight %}

또한 Heroku가 정확한 [Python Runtime](https://devcenter.heroku.com/articles/python-runtimes) 사용하기 위해서는
특정 Python version이 필요하다. 
*runtime.txt*라는 간단한 파일을 생성하고 아래의 코드를 저장하자.

{% highlight %}
python-3.5.1
{% endhighlight %}

git에 commit하고, 새로운 Heroku app을 두개 생성하라.
하나는 production 이고 : 
{% highlight %}
$ heroku create YOUR_APP_NAME (ex>wordcount-pro)
{% endhighlight %}

나머지 하나는 staging 이다 : 
{% highlight %}
$ heroku create YOUR_APP_NAME (ex>wordcount-stage)
{% endhighlight %}

> ### 이 이름들은 token 이므로, Heroku app의 이름은 unique하게 해야한다.

새로운 app을 원격 git에 추가하자. 하나는 이름을 *pro*라고 하고, 다른 하나는 *stage*라고 하자 : 

{% highlight %}
$ git remote add pro git@heroku.com:YOUR_APP_NAME.git
$ git remote add stage git@heroku.com:YOUR_APP_NAME.git
{% endhighlight %} 

이제 각 git에 push 해라 : 
 * For staging : `git push stage master`
 * For production : `git push pro master`

잘 됐다면, 브라우져에서 확인가능하다.

c.f> heroku에 ssh key 추가하기


# Staging/Production Workflow

app을 변경하고 staging에만 push 하자 : 

{% highlight % python}
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello() :
  return "Hello World"

@app.route("/<name>")
def hello_name(name) :
  return "Hello {}!".format(name)

if __name__=='__main__' : 
  app.run()

{% endhighlight %}

로컬에서 실행해보자 - `python app.py`

URL 끝에 이름을 붙여서 시도해보자 - i.e. [http://localhost:5000/moon](http://localhost:5000/moon)

이제 production에 push하기 전에 staging에서 한번 테스트해보자. staging 환경의 git에 commit과 push를 해라 - `git push stage master`

지금 stagin 환경이 동작한다면, 너는 "/moon"같은 새로운 URL를 사용할 수 있고, 브라우져에 결과물로 너가 URL에 친 name으로 "Hello Name"이라고 나올 것이다.
그러나, production site에서 같은 것을 한다면, 실패할 것이다. __즉, staging 환경에서 먼저 빌드하고 테스트 할수 있고, 그리고 나서 통과하면, production에 push 할 수 있다.__

이제 production에 push하자 - `git push pro master`

그러면 이제 동일하게 동작한다.

__이런 staging/production workflow는 production site에 아무런 변화 없이 client에게 보여주는 것과 테스트하는 것등에 변화를 줄 수 있다.__


# Config Settings

마지막으로 해야할 것은 app에 다른 config 환경을 설정하는 것이다.
우리는 자주 local, staging 그리고 production 설정을 다른게 한다. 너는 다른 데이터베이스에 접속하길 원할 수도 있고, 다른 AWS key를 사용하고 싶어할 수 있다. 다른 환경에서 config file을 다뤄보자.

project root에 *config.py* 파일을 추가하자 : 

{% highlight %}
### linux
$ touch config.py
{% endhighlight %}

{% highlight %}
### windows
>copy /b NUL config.py
{% endhighlight %}

config file은 Django의 config를 어떻게 설정하는 지에서 일부 차용했다. 
다른 config class들이 상속할 기본 config class가 있다 . 그리고 필요에 따라 적절한 config class를 import 할 것이다.

생성된 *config.py* file에 다음 코드를 추가하자 : 
{% highlight python %}
import os 
basedir = os.path.abspath(os.path.dirname(__file__))

class Config(object) :
  DEBUG = False
  TESTING = False
  CSRF_ENABLED = True
  SECRET_KEY = 'this-really-needs-to-be-changed'


class ProductionConfig(Config) :
  DEBUG = False

class StagingConfig(Config) : 
  DEVELOPMENT = True
  DEBUG = True

class DevelopmentConfig(Config) :
  DEVELOPMENT = True
  DEBUG = True

class TestingConfig(Config) :
  TESTING = True
{% endhighlight %}

`os`를 import했고 이 파일이 호출될 곳의 상대경로인 `basedir`변수를 설정했다.
그 다음, 다른 config class가 상속할 기본 설정을 가진 base `Config()` class를 정의했다. 이제 현제 환경에 맞는 적절한 config class를 import할 수 있다.
그러므로, 환경에 따라 환경변수를 선택할 수 있다 - e.g. local, staging, production.

# Local Settings

환경변수를 application에 설정하기 위해서는, [autoenv](https://github.com/kennethreitz/autoenv)을 사용해야한다.
이 프로그램은 디렉토리에 이동할 때마다 실행되는 command를 설정할 수 있게 한다.
이것을 사용하기 위해서는, 이것을 글로벌로 설치해야한다. 
먼저, 터미널의 가상환경에서 나오고, autoenv를 설치하고 *.env* file을 추가하라 : 
{% highlight %}
deactivate
pip install autoenv==1.0.0
touch .env
{% endhighlight %}

그러고나서, *.env* file에 다음을 추가하라 : 

{% highlight %}
source env/bin/active
export APP_SETTINGS="config.DevelopmentConfig"
{% endhighlight %}

업데이트를 위해 다음을 실행하고 *.bashrc*를 refresh해라 :

{% highlight %}
echo "source 'which activate.sh'" >> ~/.bashrc
source ~/.bashrc
{% endhighlight %}

이제 디렉토리를 이동했다 돌아오면, 가상환경은 자동적으로 시작되고 `APP_SETTINGS`변수가 선언된다.

# Heroku Settings

Heroku에도 비슷하게 환경변수를 설정한다.

staging을 위해 아래의 command를 실행한다 : 
{% highlight %}
$ heroku config:set APP_SETTINGS=config.StagingConfig --remote stage
{% endhighlight %}

production을 위해서는 : 

{% highlight %}
$ heroku config:set APP_SETTINGS=config.ProductionConfig --remote pro
{% endhighlight %}

맞는 환경을 사용할 수 있게 *app.py*를 변경하라 : 
{% highlight python %}
import os
from flask import Flask

app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])

@app.route("/")
def hello():
  return "Hello world"

@app.route("/<name>")
def hello_name(name) :
  return "Hello {}!".format(name)

if __name__ == '__main__' : 
  app.run()

{% endhighlight %}

환경에 맞는 적절한 `APP_SETTINGS`변수를 import하기 위해 `os`를 import 하고 `os.environ` method를 사용했다.
그리고 나서 `app.config.from_object` method로 app의 config를 설정하였다.

staging과 production 양 쪽 모두 commit하고 push해라.

환경에 따라 환경변수가 맞게 설정되는지 확인하고 싶다면 *app.py*에 다음을 추가하라 : 
{% highlight python %}
print(os.environ['APP_SETTINGS'])
{% endhighlight %}

이제 app을 실행하면, config 설정값을 확인할 수 있다.

__Local__ : 
{% highlight %}
$ python app.py
config.DevelopmentConfig
{% endhighlight %}

staging과 production에도 commit하고 push해라. 이제 테스트 해보면

__Staging__ : 
{% highlight python%}
$ heroku run python app.py --app wordcount-stage
Running python app.py on wordcount-stage... up, run.7699
config.StagingConfig
{% endhighlight %}

__Production__ :
{% highlight %}
$ heroku run python app.py --app wordcount-pro
Running python app.py on wordcount-pro... up, run.8934
config.ProductionConfig
{% endhighlight %}

테스트가 끝나면 `print(os.environ['APP_SETTINGS'])`를 제거하고, 모든 환경에 commit과 push를 하라.

[원문 : flask-by-example-part1](https://realpython.com/blog/python/flask-by-example-part-1-project-setup/)