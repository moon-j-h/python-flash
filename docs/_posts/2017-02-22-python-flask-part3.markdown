---
layout: post
title:  "part3"
date:   2017-02-22 21:26:47 +0900
categories: translation
---

이 포스팅은 [원문](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/)을 의역하고 필요한 부분은 추가한 것.


> Part three : back-end에 scrape을 위한 로직을 추가하고, requests, BeautifulSoup, 과 Natural Language Toolkit(NLTK) library를 사용하여 webpage에서 단어 수를 세는 것을 해 보자

# Install Requirements

이 part에 사용되는 tool 들을 확인하자

 - [requests](https://pypi.python.org/pypi/requests/2.13.0) - HTTP request 를 보내기 위한 library
 - [BeautifulSoup](https://pypi.python.org/pypi/beautifulsoup4/4.5.3) - 웹의 문서를 scraping하고 parsing하기 위한 tool
 - [Natural Language Toolkit](https://pypi.python.org/pypi/nltk/3.2.2) - library를 처리하기 위한 natural language
 
virtual environment 을 활성화하기 위해 디렉토리로 이동하고, 위의 tool들을 설치하라

```python
$ cd flask-by-example
$ pip install requests beautifulsoup4 nltk
$ pip freeze > requirements.txt
```
# Refactor the Index Route

시작하기 위해, *app.py* 파일의 index route의 "hello world" 부분을 제거하고 URL을 받는 form을 render하는 route를 설정하라.
먼저, template을 저장할 **templats** 폴더를 추가하고 *index.html*을 추가하라.

```python
$ mkdir templates

### linux
$ touch templates/index.html 
### windows
> cd templates
> copy /b NUL index.html
```
기본적인 HTML page을 추가하라.
```html
<!DOCTYPE html>
<html>
    <head>Wordcount</head>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="//netdna.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css" rel="stylesheet" media="screen">
    <style>
      .container {
        max-width: 1000px;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h1>Wordcount 3000</h1>
      <form role="form" method='POST' action='/'>
        <div class="form-group">
          <input type="text" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;" autofocus required>
        </div>
        <button type="submit" class="btn btn-default">Submit</button>
      </form>
      <br>
      {% for error in errors %}
        <h4>{{ error }}</h4>
      {% endfor %}
    </div>
    <script src="//code.jquery.com/jquery-2.2.1.min.js"></script>
    <script src="//netdna.bootstrapcdn.com/bootstrap/3.3.6/js/bootstrap.min.js"></script>
  </body>
</html>
```
최소한의 style을 주기 위해 bootstrap을 사용했다. 그리고 사용자가 URL을 입력할 text input을 추가했다.
거기다, `for` loop를 사용하기 위해 [Jinja](https://realpython.com/blog/python/primer-on-jinja-templating/)를 확용했다.

template를 제공하게 *app.py*를 수정하자

```python
import os
from flask import Flask, render_template
from flask.ext.sqlalchemy import SQLAlchemy 

app = Flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

from models import Result

@app.route('/', methods=['GET', 'POST'])
def index():
    return render_template('index.html')

if __name__=='__main__':
    app.run()
```
왜 `method=['GET', 'POST']`에서 둘다 사용했을까? *index.html* 페이지를 제공하고 각 form submit을 다루기 위해서 GET, POST둘다 사용한다.

테스트를 위해 아래를 작성하라
```python
$ python manage.py runserver
```

[http://localhots:5000](http://localhots:5000) 에 들어가면 확인할 수 있다.

# Requests

해당 URL의 HTML page를 가져오기 위해 requests library를 사용하자.
index route을 다음과 같이 변경해라:
```python
@app.route('/', methods=['GET', 'POST'])
def index():
    errors = []
    results = {}
    if request.method == "POST":
        try :
            url = request.form['url']
            r = requests.get(url)
            print(r.text)
        except:
            errors.append("Unable to get URL. Please make sure it's valid and try again.")
    return render_template('index.html', errors=errors, results=results)
```
다음과 같이 imports가 수정된것을 확인하라:
```python
import os
import requests
from flask import Flask, render_template, request
from flask.ext.sqlalchemy import SQLAlchemy
```
1. Flask의 `request` 뿐만 아니라 `requests` library도 import 했다. 
전자는 Flask app 내부의 GET과 POST를 다루기 위한 것인 반면, 후자는 사용자가 제공한 특정 URL을 잡기 위해 외부의 HTTP GET requests를 보내는 것에 사용된다.
2. template에 넘겨주는 errors 와 results를 저장하기 위한 변수를 추가했다.
3. view내부에 request가 GET인지 POST인지 확인할 수 있다.
    - POST : form에서 url 갑을 가져오고 `url` 변수에 저장한다. 그리고 errors을 다루기 위한 exception을 추가하고, 필요하다면, 일반적인 error message를 `errors` list에 추가한다.
    마지막으로,  `errors` list와 `results` dictionary를 포함한 template을 rendering 한다.
    - GET : 단순히 template을 rendering한다.

이제 테스트해보자:
```python
$ python manage.py runserver
``` 

유효한 webpage을 입력하면, return된 page의 text를 터미널에서 볼 수 있다.

# Text Processing

HTML에서 페이지에 있는 단어의 빈도수를 세고 user에게 보여준다.
*app.py* code를 수정하고 무슨 일이 일어나는지 확인하자
```python
import os 
import requests
import operator
import re
import nltk
from flask import Flask, render_template, request
from flask.ext.sqlalchemy import SQLAlchemy
from stop_words import stops
from collections import Counter
from bs4 import BeautifulSoup

app = flask(__name__)
app.config.from_object(os.environ['APP_SETTINGS'])
app.config.['SQLALCHEMY_TRACK_MODIFICATIONS'] = True
db = SQLAlchemy(app)

from models import Result

@app.route('/', methods=['GET', 'POST'])
def index():
  errors = []
  results = {}
  if request.method=="POST":
    try:
      url = request.form['url']
      r = requests.get(url)
    except:
      errors.append("Unable to get URL. Please make sure it's valid and try again.")
      render_template('index.html', errors=errors)
    if r:
      # text processing
      raw = BeautifulSoup(r.text, 'html.parser').get_text()
      nltk.data.path.append('./nltk_data/') # set the path
      tokens = nltk.word_tokenize(raw)
      text = nltk.Text(tokens)
      # 구두점 제거하기, raw words 세기
      nonPunct = re.compile('.*[A-Za-z].*')
      raw_words = [w for w in text if nonPunct.match(w)]
      raw_word_count = Counter(raw_words)
      #stop words
      no_stop_words = [w for w in raw_words if w.lower() not in stops]
      no_stop_words_count = Counter(no_stop_words)
      # 결과 저장하기
      results = sorted(
        no_stop_words_count.items(),
        key=operator.itemgetter(1),
        reverse=True
      )
      try:
        result = Result(
          url=url,
          result_all=raw_word_count,
          result_no_stop_words=no_stop_words_count
        )
        db.session.add(result)
        db.session.commit()
      except:
        errors.append("Unable to add item to database.")
    return render_template('index.html', errors=erros, results=results)
  
  if __name__=='__main__':
    app.run()
```

*stop_words.py*라는 새로운 파일을 만들고 아래의 것을 저장하라:
```python
stops=[
  'i', 'me', 'my', 'myself', 'we', 'our', 'ours', 'ourselves', 'you',
    'your', 'yours', 'yourself', 'yourselves', 'he', 'him', 'his',
    'himself', 'she', 'her', 'hers', 'herself', 'it', 'its', 'itself',
    'they', 'them', 'their', 'theirs', 'themselves', 'what', 'which',
    'who', 'whom', 'this', 'that', 'these', 'those', 'am', 'is', 'are',
    'was', 'were', 'be', 'been', 'being', 'have', 'has', 'had', 'having',
    'do', 'does', 'did', 'doing', 'a', 'an', 'the', 'and', 'but', 'if',
    'or', 'because', 'as', 'until', 'while', 'of', 'at', 'by', 'for',
    'with', 'about', 'against', 'between', 'into', 'through', 'during',
    'before', 'after', 'above', 'below', 'to', 'from', 'up', 'down', 'in',
    'out', 'on', 'off', 'over', 'under', 'again', 'further', 'then',
    'once', 'here', 'there', 'when', 'where', 'why', 'how', 'all', 'any',
    'both', 'each', 'few', 'more', 'most', 'other', 'some', 'such', 'no',
    'nor', 'not', 'only', 'own', 'same', 'so', 'than', 'too', 'very', 's',
    't', 'can', 'will', 'just', 'don', 'should', 'now', 'id', 'var',
    'function', 'js', 'd', 'script', '\'script', 'fjs', 'document', 'r',
    'b', 'g', 'e', '\'s', 'c', 'f', 'h', 'l', 'k'
]
```

# What's happening?

#### Text Processing

1. index route에서 URL에서 가져온 HTML tag를 제거하는 등, text를 깨끗이 하기 위해 beautifulsoup을 사용했다.
  - [Tokenize](http://www.nltk.org/api/nltk.tokenize.html) text를 각 단어로 쪼갰고
  - Turn the tokens into a nltk [text object](http://www.nltk.org/_modules/nltk/text.html)
2. nltk를 단어에 적절하게 사용하기 위해서는, 맞는 [tokenizers](http://www.nltk.org/api/nltk.tokenize.html#module-nltk.tokenize.punkt)를 다운받아야 한다.
먼저, 새로운 디렉토리를 생성하고 - `mkdir nltk_data`, `python -m nltk.downloader`을 실행하라.
설치 창이 나타나면, whatever_the_absolute_path_to_your_app_is/nltk_data인 'Download Directory'에 업데이트해라.
그리고 'Models' tab을 클릭하고 'Identifire'컬럼 아래의 'punkt'를 선택하라. 'Download'를 클릭하라. 
더 정보를 원한다면 [공식문서](http://www.nltk.org/data.html#command-line-installation)를 확인하라.

#### Remove Punctuation, Count Row words

1. 마지막 결과에서 구두점을 세지않기 원한다면, 우리는 알파벳에 매칭되는 정규표현식을 만들어야한다.
2. 그리고, 구두점이나 숫자가 포함되지 않은 단어들의 리스트를 만들어서 사용한다.
3. 마지막으로, [Counter](https://pymotw.com/3/collections/counter.html)를 사용하여 리스트에 각 단어가 몇번 나왔는지 추가하라.

#### Stop Words

현재의 output은 우리가 원하지 않은 많은 단어도 포함하고 있다. - "I", "me" "the"와 같은. 이러한 단어들을 stop words 라고 한다.

1. `stops` 리스트를 사용해서, 우리는 다시 이러한 stop words를 포함하지 않은 리스트를 만든다.
2. 그 다음, 우리는 단어를 key로, 그 수를 value로 하는 word dictionary를 만든다.
3. 그리고 마지막으로 dictionary를 정렬하기 위해  [sorted](https://docs.python.org/3/library/functions.html#sorted) method를 사용한다.
이제 우리는 가장 많은 단어 순으로 display할 수 있게 정렬된 데이터를 사용할 수 있다.이는 Jinja template 에서 따로 정렬할 필요가 없다는 것을 의미한다.

> 좀 더 자세한 stop word list를 원한다면, NLTK [stopwords corpus](http://www.nltk.org/book/ch02.html) 를 사용하라

#### Save the Results

마지막으로, 검색하고 count한 결과를 데이터베이스에 저장하기 위해 try/except 문을 사용한다.

# Display Results 

결과를 display하기 위해 *index.html*을 수정하자.

```xml
<!DOCTYPE html>
<html>
  <head>
    <title>Wordcount</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css" rel="stylesheet" media="screen">
    <style>
      .container {
        max-width: 1000px;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <div class="row">
        <div class="col-sm-5 col-sm-offset-1">
          <h1>Wordcount 3000</h1>
          <br>
          <form role="form" method="POST" action="/">
            <div class="form-group">
              <input type="text" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;">
            </div>
            <button type="submit" class="btn btn-default">Submit</button>
          </form>
          <br>
          {% for error in errors %}
            <h4>{{ error }}</h4>
          {% endfor %}
          <br>
        </div>
        <div class="col-sm-5 col-sm-offset-1">
          {% if results %}
            <h2>Frequencies</h2>
            <br>
            <div id="results">
              <table class="table table-striped" style="max-width: 300px;">
                <thead>
                  <tr>
                    <th>Word</th>
                    <th>Count</th>
                  </tr>
                </thead>
                {% for result in results%}
                  <tr>
                    <td>{{ result[0] }}</td>
                    <td>{{ result[1] }}</td>
                  </tr>
                {% endfor %}
              </table>
            </div>
          {% endif %}
        </div>
      </div>
    </div>
    <br><br>
    <script src="//code.jquery.com/jquery-1.11.0.min.js"></script>
    <script src="//netdna.bootstrapcdn.com/bootstrap/3.1.1/js/bootstrap.min.js"></script>
  </body>
</html>
```
여기, 우리는 `results` dictionary에 저장된 것을 있을 때만 보기 위해 `if`문을 추가했다.
그리고 `results`를 iterate해서 테이블로 보여주기 위해 `for` loop를 추가했다. app을 실행해라 그리고 URL을 입력하면 페이지에 결과나 나타날 것이다.

```python
$ python manage.py runserver
```

10개의 단어만 나타나게 하려면 아래와 같이 하면 된다.

```python
results = sorted(
  no_stop_words_count.item(),
  key=operator.itemgetter(1),
  reverse=True
)[:10]
```

시도해봐라.



[원문 : flask-by-example-part3](https://realpython.com/blog/python/flask-by-example-part-3-text-processing-with-requests-beautifulsoup-nltk/)