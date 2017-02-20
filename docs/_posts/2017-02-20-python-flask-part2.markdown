---
layout: post
title:  "part2"
date:   2017-02-20 21:26:47 +0900
categories: translation
---

이 포스팅은 [원문](https://realpython.com/blog/python/flask-by-example-part-2-postgres-sqlalchemy-and-alembic/)을 의역하고 필요한 부분은 추가한 것.


> Part two : SQLAlchemy의 PostgreSQL database 과 migration을 위한 Alembic 을 설치하자

# Install Requirements

이 part에 사용되는 tool 들을 확인하자

 - [PostgreSQL](https://www.postgresql.org/)
 - [Psycopg2](https://pypi.python.org/pypi/psycopg2) `pip install psycopg2`
 - [Flask-SQLAlchemy](http://flask-sqlalchemy.pocoo.org/2.1/) `pip install flask-sqlalchemy`
 - [Flask-Migrate]( https://devcenter.heroku.com/articles/getting-started-with-python) `pip install flask-migrate`


시작하기 앞서, 로컬에 Postgre을 설치하자. Heroku에서 postgres을 사용한 후부터, 같은 database에서 local 개발할 때 유용하다.

Postgres 가 잘 설치되고 실행되면, local 개발 database로 사용할 `wordcount_dev`라는 database를 생성해라.

# Update Configuration

local, staging, 그리고 production에서 새롭게 생성한 database을 사용하기 위해 *config.py* 파일의 `Config()` class에 `SQLALCHEMY_DATABASE_URI`를 추가하라

```python
import os

class Config(Object):
  ...
  SQLALCHEMY_DATABASE_URI = os.environ['DATABASE_URL']
```

*config.py* 는 전체적으로 이런 모습이다.
```python
import os
basedir = os.path.abspath(os.path.dirname(__file__))

class Config(Object):
  DEBUG = False
  TESTING = False
  CSRF_ENABLED = True
  SECRET_KEY = 'this-really-needs-to-be-changed'
  SQLALCHEMY_DATABASE_URI = os.environ['DATABASE_URL']

class ProductionConfig(Config):
  DEBUG = False

class StagingConfig(Config):
  DEVELOPMENT = True
  DEBUG = True

class DevelopmentConfig(Config):
  DEVELOPMENT = True
  DEBUG = True

class TestingConfig(Config):
  Testing = True
```
이제 config가 로드될때 적절한 database를 잘 연결할 것이다.

지난 포스팅에 환경변수를 추가한 것처럼 `DATABASE_URL` 변수를 추가하자.

```php
$ export DATABASE_URL="postgresql://localhost/wordcount_dev"
```
그리고 *.env* 파일에 이 줄을 추가하자.

*app.py* 파일에서 SQLAlchemy 를 import하고 database를 연결한다.
```python
from flask import Flask
from flask.ext.sqlalchemy import SQLAlchemy
import os

app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTING'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

from models import Result

@app.route('/')
def hello():
  return "Hello World!"

@app.route('/<name>')
def hello_name(name):
  return "Hello {}!".format(name)

if __name__=='__main__':
  app.run()
```

# Data models

*models.py* 파일을 추가해서 기본적인 model 을 세팅하자

```python
from app import db
from sqlalchemy.dialects.postgresql import JSON

class Result(db.Model):
  __tablename__='results'

  id = db.Column(db.Integer, primary_key=True)
  url = db.Column(db.String())
  result_all = db.Column(JSON)
  result_no_stop_words = db.Column(JSON)

  def __init__(self, url, result_all, result_no_stop_words):
    self.url = url
    self.result_all = result_all
    self.result_no_stop_words = result_no_stop_words

  def __repr__(self):
    return '<id {}>'.format(self.id)
```
*counts* 라는 단어의 결과를 저장하는 table을 생성했다.

SQLAlchemy의 PostgreSQL dialects의 JSON 뿐만 아니라, *app.py* 파일에서 생성했던 database connection도 가장 먼저 import 한다.
JSON 칼럼은 Postgres에서 처음 사용하는 것이고 SQLAlchemy에서 지원하는 모든 database에서 이용할 수 없다. 그래서 특별히 이것을 import 해주어야한다.

다음으로, 우리는 `Result()` class를 생성하고 table 이름인 `results`를 부여한다. 그리고 result에 저장하고 싶은 것들을 attribute로 설정한다.

 - `id`
 - `url` : 어디서 연결되었는지
 - 우리가 count한 단어들의 list
 - 우리가 count한 minis stop words의 list

그러고 나서, 새로운 result가 생성된 때 처음 실행하는 `__init__()` method를 생성하고 query를 날릴때 obejct를 대신하는 `__repr()__` method 생성한다.

# Local Migration

database의 schema를 업데이트하기 위해 database migration을 관리하는 **Flask-Migrate**의 일부분인 **Alembic**을 사용할 것이다.

*manange.py*라는 파일을 생성한다.
```python
import os
from flask.ext.script import Manager
from flask.ext.migrate import Migrate, MigrateCommand

from app import app, db

app.config.from_object(os.environ['APP_SETTINGS'])

migrate = Migrate(app, db)
manager = Manager(app)

manager.add_command('db', MigrateCommand)

if __name__=='__main__':
  manager.run()
```

Flask-Migrate 를 사용하기 위해서는, *manage.py* 파일에 `Migrate`와 `MigrateCommand`뿐만 아니라 `Manager` 또한 import해준다.
또한 `app`과 `db`도 import해주어 script안에서 접근할 수 있게 한다.

먼저, -환경 변수에 기반하는- environment에서 config를 세팅하고 
`app`과 `db`를 argument로 하는 migrate instance를  생성하며, 마지막으로 `manager`에 `db` command를 추가하면 command line에서 migration을 실행할 수 있다.

Alembic으로 초기설정한 migrations을 실행하기 위해서는
```php
$ python manage.py db init
  Creating directory /flask-by-example/migrations ... done
  Creating directory /flask-by-example/migrations/versions ... done
  Generating /flask-by-example/migrations/alembic.ini ... done
  Generation /flask-by-example/migrations/env.py ... done
  Generating /flask-by-example/migrations/README ... done
  Generating /flask-by-example/migrations/script.py.mako ... done
  Please edit configuration/connection/logging settings in
  '/flask-by-example/migrations/alembic.ini' before proceeding.
```

database initialization을 실행 한 후, "migrations"라는 새로운 폴더를 볼 수 있다.
이것은 Alembic이 project에 대항하여 migration을 실행할 때 필요한 설정정보가 있다.
"migrations"내부에는 생성된 migration script가 포함된 "versions"라는 폴더를 볼 수 있다.

`migrate` command 실행을 통해 처음 migration을 생성해보자.

```php
$ python manage.py db migrate
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.autogenerate.compare] Detected added table 'results'
    Generating /flask-by-example/migrations/versions/63dba2060f71_.py
    ... done
```
이제 "version"폴더에 migration 파일을 확인할 수 있다. 이 파일은 model 기반으로 Alembic이 자동생성한다. 스스로 생성하거나 수정할 수도 있다.
그러나 대부분은 자동생성을 한다.

다음은 `db upgrade` command를 사용해 database에 수정사항을 적용하자.

```php
$ python manage.py db upgrade
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.runtime.migration] Running upgrade  -> 63dba2060f71, empty message
```
database는 우리 앱에 사용할 준비가 된다.

```php
$ psql
# \c wordcount_dev
You are now connected to database "wordcount_dev" as user "michaelherman".
# \dt

                List of relations
 Schema |      Name       | Type  |     Owner
--------+-----------------+-------+---------------
 public | alembic_version | table | michaelherman
 public | results         | table | michaelherman
(2 rows)

# \d results
                                     Table "public.results"
        Column        |       Type        |                      Modifiers
----------------------+-------------------+------------------------------------------------------
 id                   | integer           | not null default nextval('results_id_seq'::regclass)
 url                  | character varying |
 result_all           | json              |
 result_no_stop_words | json              |
Indexes:
    "results_pkey" PRIMARY KEY, btree (id)
```
# Remote Migration

마지막으로, Herohu의 database에 migration을 적용해보자.
먼저, *config.py* 파일에 staging과 production의 database에 관한 세부정보를 추가할 필요가 있다.

staging server에 database가 설치되어 있는지 확인하기 위해 
```php
$ heroku config --app wordcount-stage
=== wordcount-stage Config Vars
APP_SETTINGS: config.StagingConfig
```

> ### `wordcount-stage`는 너의 staging app의 이름에 맞게 수정하라

database의 환경변수를 확인할 수 없어진 이래로, staging server에 **Postgres addon**을 추가할 필요가 있다.

```php
$ heroku addons:create heroku-postgresql:hobby-dev --app wordcount-stage
  Creating postgresql-cubic-86416... done, (free)
  Adding postgresql-cubic-86416 to wordcount-stage... done
  Setting DATABASE_URL and restarting wordcount-stage... done, v8
  Database has been created and is available
   ! This database is empty. If upgrading, you can transfer
   ! data from another database with pg:copy
  Use `heroku addons:docs heroku-postgresql` to view documentation.
```
> ### `hobby-dev`는 Heroku Postgres addon의 [free tier](https://elements.heroku.com/addons/heroku-postgresql#hobby-dev) 이다

이제 `heroku config --app wordcount-stage`를 실행할 때 database의 connect 설정도 확인할 수 있다.

```php
=== wordcount-stage Config Vars
APP_SETTINGS: config.StagingConfig
DATABASE_URL: postgres://azrqiefezenfrg:Zti5fjSyeyFgoc-U-yXnPrXHQv@ec2-54-225-151-64.compute-1.amazonaws.com:5432/d2kio2ubc804p7
```

다음으로 수정된 것을 staging server에 commit하고 push해라

```php
$ git push stage master
```

`heroku run` command를 사용하여 staging database를 migrate하기 위해 생성한 migrations를 실행하라

```php
$ heroku run python manage.py db upgrade --app wordcount-stage
  Running python manage.py db upgrade on wordcount-stage... up, run.5677
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  INFO  [alembic.runtime.migration] Running upgrade  -> 63dba2060f71, empty message
```

> ### `upgrade`만 실행했다. `init`와 `migrate`는 전에 하지 않았다. 우리는 이미 설정된 migration 파일을 가지고 있고, 할 준비가 되어있다.; 단지 Heroku database에 맞게 적용할 필요가 있다????

이제 production에도 똑같이 한다.

 1. Heroku의 production app에 database를 설정하자. : `heroku addons:create heroku-postgresql:hobby-dev --app wordcount-pro`
 2. 변경사항을 production에 push 해라 : `git push pro master` config 파일을 수정할 필요가 없다는 것에 주목해라 - 새롭게 생성된 `DATABASE_URL` 환경변수를 바탕으로 database가 설정된다.
 3. migration을 적용하자 : `heroku run python manage.py db upgrade --app wordcount-pro`

이제 staging과 production 둘다 database가 설정되었고 migrate 할 준비가 되었다.

> ### production database에 새로운 migration을 적용할 때, site가 down될 수도 있다. 이것이 issue라면, "follower"(보통 slave로 알려짐)을 추가하여 database replication을 설정할 수 있다. 이것에 대한 정보는, [공식 heroku 문서](https://devcenter.heroku.com/articles/heroku-postgres-follower-databases)를 확인하라



[원문 : flask-by-example-part2](https://realpython.com/blog/python/flask-by-example-part-2-postgres-sqlalchemy-and-alembic/)