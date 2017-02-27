---
layout: post
title:  "part4"
date:   2017-02-24 18:16:21 +0900
categories: translation
---

이 포스팅은 [원문](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/)을 의역하고 필요한 부분은 추가한 것.


> Part four : text processing을 다루기 위해 Redis task queue를 실행하라

# Install Requirements

이 part에 사용되는 tool 들을 확인하자

 - [Redis](https://redis.io/)
 - [Python Redis](https://pypi.python.org/pypi/redis/2.10.5)
 - [RQ](https://pypi.python.org/pypi/rq/0.5.6) - task queue를 생성하기 위한 간단한 library
 
Redis를 설치하라. 설치가 끝나면 Redis server를 실행하라

```python
$ redis-server 
```
다음 Python Redis와 RQ를 설치하라

```python
$ cd flask-by-example
$ pip install redis rq
$ pip freeze > requirements.txt
```
# Set up the Worker

queued task를 listen하기 위한 worker process를 생성하자. *worker.py* 파일을 생성하고 다음의 코드를 추가하라.

```python
import os 
import redis
from rq import worker, Queue, Connection

listen = ['default']

redis_url = os.getenv('REDISTOGO_URL', 'redis://localhost:6379')

conn = redis.from_url(redis_url)

if __name__ == '__main__':
  with Connection(conn):
    worder = Worker(list(map(Queue, listen)))
    worker.work()
```

위 코드에서는, `default`라는 queue를 listen하고 있다. 그리고 `localhost:6379`에서 Redis server 에 connection를 생성한다.

다른 terminal을 열어서 아래를 실행하라
```python
$ cd flask-by-example
$ python worker.py
17:01:29 RQ worker started, version 0.5.6
17:01:29
17:01:29 *** Listening on default...
```
이제 우리는 queue에 job을 보내기 위해 *app.py* 을 업데이트해야한다.

# Update app.py

다음의 import들을 *app.py*에 추가하라:
```python
from rq import Queue
from rq.job import Job
from worker import conn
```

configuration section도 업데이트 해라:
```python
app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
db = SQLAlchemy(app)

q = Queue(connection=conn)

from models import *
```

`q = Queue(connection=conn)` 은 Redis connection을 설정하고 connection을 기반으로 하는 queue를 초기화한다.

text processing 기능을 index route의 밖으로 분리하고 `count_and_save_words()`라는 새로운 function을 만들어 호출한다.
이 function은 index route에서 호출될 때, URL이라는 한개의 argument을 받는다.

```python
def count_and_save_words(url):
  errors = []

  try:
    r = requests.get(url)
    except:
    errors.append(
      "불가능한 URL. 유효한 URL인지 확인하고 다시 시도하세요."
    )
    return {"error":errors}
  #text processing
  raw = BeautifulSoup(r.text).get_text()
  nltk.data.path.append('./nltk_data/')
  tokens = nltk.word_tokenize(raw)
  text = nltk.Text(tokens)

  #remove punctuation, count raw words
  nonPunct = re.compile('.*[A-Za-z].*')
  raw_words = [w for w in text if nonPunct.match(w)]
  raw_word_count = Counter(raw_words)

  #stop words
  no_stop_words = [w for w in raw_words if w.lower() not in stops]
  no_stop_words_count = Counter(no_stop_words)

  #save the results
  try:
    result = Result(
      url=url,
      result_all=raw_word_count,
      result_no_stop_words=no_stop_words_count
    )
    db.session.add(result)
    db.session.commit()
    return result.id
  except:
    errors.append("database에 result 추가 실패")
    return {"error" : errors}

@app.route('/', methods=['GET', 'POST'])
def index():
  results={}
  if request.method == "POST":
    url = request.form['url']
    if 'http://' not in url[:7]:
      url = 'http://'+url
    job = q.enqueue_call(
      func=count_and_save_words, args=(url,), result_ttl=5000
    )
    print(job.get_id())
  
  return render_template('index.html', results=results)
```

다음의 코드에 주목해라:
```python
job = q.enqueue_call(
  func = count_and_save_words, args=(url,), result_ttl=5000
)
print(job.get_id())
```

전에 초기화한 queue을 사용하고 `enqueue_call()` function을 호출한다. 이는 queue에 새로운 job을 추가하고
argument로 URL을 받은 `count_and_save_words()` function을 실행한다. 
`result_ttl=5000`이라는 argument는 RQ가 job의 결과를 얼마까지 hold할 것인지를 나타낸다.- 위 경우는 5000 초. 
그러고 나면 우리는 job id를 terminal에 출력한다. 이 id는 job이 processing을 끝내면 보기 위해 필요하다.
이제 새로운 route를 설정하자...

# Get Results 

```python
@app.route('/results/<job_key>', methods=['GET'])
def get_results(job_key):

  job = Job.fetch(job_key, connection=conn)

  if job.is_finished:
    return str(job.result), 200
  else:
    return "Nay!", 202
```
이제 테스트 해보자
서버를 실행하고 [http://localhost:5000/](http://localhost:5000/)으로 가고,[http://realpython.com](http://realpython.com) URL을 사용하고, teminal에서 job id를 확인하라.
그러고 나서 그 id를 '/results/'의 끝에 사용하라 - i.e.[http://localhost:5000/results/ef600206-3503-4b87-a436-ddd9438f2197](http://localhost:5000/results/ef600206-3503-4b87-a436-ddd9438f2197)

5000초보다 시간이 많이 걸리면 database에 results를 저장할 때 생선한 id를 다시 확인해야한다.
```python
# save the results
try:
  from models import Result
  result = Result(
    url=url,
    result_all = raw_word_count,
    result_no_stop_words = no_stop_words_count
  )
  db.session.add(result)
  db.session.commit()
  return result.id
```

이제 route를 database에서 찾은 results를 JSON으로 return하도록 약간 수정하자.

```python
@app.route("/results/<job_key>", methods=["GET"])
def get_results(job_key):
  job = Job.fetch(job_key, connection=conn)

  if job.is_finished:
    result = Result.query.filter_by(id=job.result).first()
    results = sorted(
      result.result_no_stop_words.items(),
      key = operator.itemgetter(1),
      reverse = True
    )[:10]
    return jsonify(results)
  else:
    return "Nay!", 202
```

import를 추가하는 것을 잊지말자:

```python
from flask import jsonify
```

다시 test해보자. 다 잘했다면, 다음과 같은 결과가 browser에 나타날 것이다.
```python
{
  Course: 5,
  Python: 19,
  Real: 11,
  course: 4,
  courses: 7,
  development: 7,
  product: 4,
  sample: 4,
  videos: 5,
  web: 12
}
```


[원문 : flask-by-example-part4](https://realpython.com/blog/python/flask-by-example-implementing-a-redis-task-queue/)