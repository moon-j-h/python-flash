---
layout: post
title:  "part5"
date:   2017-02-25 20:31:47 +0900
categories: translation
---

이 포스팅은 [원문](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/)을 의역하고 필요한 부분은 추가한 것.


> Part five : request가 끝나면 지속적으로 back-end에 poll 하기 위한 front-end를 Angular로 설정하자

# Current Functionality

먼저, terminal에 Redis를 실행시켜라.

```python
$ redis-server
```

다른 창에 project 디렉토리로 이동하여 worker를 실행하라:

```python
$ cd flask-by-example
$ python worker.py
20:38:04 RQ worker started, version 0.5.6
20:38:04
20:38:04 *** Listening on default...
```
마지막으로, 3번째 terminal를 열고, project 디렉토리로 이동하여, main app을 실행하라:

```python
$ cd flask-by-example
$ python manage.py runserver
```

[http://localhost:5000](http://localhost:5000)을 열고 [http://realpython.com](http://realpython.com) URL를 test해라. 
terminal에 job id가 출력될 것이다. 이 id를 이용하여 다음의 url로 이동하라:

[http://localhost:5000/results/add_the_job_id_here](http://localhost:5000/results/add_the_job_id_here)

다음과 비슷한 JSON response를 browser에서 볼 수 있다.

```python
{
  Course: 5,
  Python: 19,
  Real: 11,
  course: 4,
  courses: 7,
  development: 7,
  sample: 4,
  three: 4,
  videos: 5,
  web: 12
}
```

이제 우리는 Angular를 추가할 준비가 됐다.

# Update index.html

*index.html*에 Angular를 추가하자:

```html
<script src="//ajax.googleapis.com/ajax/libs/angularjs/1.4.9/angular.min.js"></script>
```

*index.html*에 다음의 [directives](https://code.angularjs.org/1.4.9/docs/guide/directive)를 추가하라:

1. ng-app : `<html ng-app="WordcountApp">`
2. ng-controller : `<body ng-controller="WordcountController">`
3. ng-submit : `<form role="form" ng-submit="getResults()">`

그래서, 우리는 Angular application으로서 HTML 문서를 다루는 Angular를 호출하는 Angular를 로딩하고, 
controller를 추가하고,form submit에서 trigger되는 `getReaults()`라는 function을 추가한다.

# Create The Angualr Module

"static" directory를 생성하고 *main.js*라는 파일을 directory에 추가하라. 
그리고 이 파일을 *index.html*에 추가하라 : 
`<script src="{{ url_for('static', filename='main.js')}}"></script>`

기본 코드를 작성하자:
```javascript
(function(){
  'use strict':
  angular.module('WordcountApp', [])
  .controller('WordcountController', ['$scope', '$log', 
    function($scope, $log){
      $scope.getResults = function(){
        $log.log("test");
      };
    }
  ]);
}());
```
이제, form이 submit 될 때, browser의 JavaScript console에 간단하게 "test"라고 log를 남기는 `getResults()`를 호출한다. 
test해서 확인하라.

# Dependency Injection and $scope

위의 예제에서, 우리는 `$scope` obejct 와 `$log` service를 "inject"하기 위해
[dependency injection](https://code.angularjs.org/1.4.9/docs/guide/di)을 활용했다.
여기서 멈춰라. `$scope`을 이해하는 것은 매우 중요하다. [Angular document](https://code.angularjs.org/1.4.9/docs/guide/scope)확인하라.
그리고, 아직 모르겠다면 [Angular intro tutorial](https://github.com/mjhea0/thinkful-angular)을 해봐라.

`$scope`은 복잡할 것 같지만, 단지 View와 Controller간의 communication을 제공한다.둘다 이것에 접근 할 수 있고, 
한쪽에서 `$scope`에 연결된 변수를 변경하면 변수는 자동적으로 다른 쪽에도 업데이트 된다.([data binding](https://code.angularjs.org/1.4.9/docs/guide/databinding)).
같은 말로 dependency injection 이라고 한다: 이것은 말보다 간단하다. 이것은 단순히 다양한 서비스에 접근하기 위해 존재해는 마법의 일종이라고 생각하라.
그래서 service inject를 통해, 우리는 지금 controller에서 사용할 수 있다.

이제 우리 app을 돌아가서..

test를 한다면, 너는 POST request를 back-end로 보내는 것이 아닌 form submission을 볼 것이다. 이것이 우리가 원하는 것이다.
대신에, 우리는 Angular의 `$http` service를 사용할 것이다.

```Javascript
.controller("WordCountroller", ['$scope', '$log', '$http', 
  function($scope, $log, $http){
    $scope.getResults = function(){
      $log.log("test");
      var userInput = $scope.url;
      $http.post("/start", {"url" : userInput})
      .success(function(results){
        $log.log(results);
      })
      .error(function(err){
        $log.log(err);
      });
    };
}]);
```
또한, *index.html*의 `input` element도 업데이트하라:
```html
<input type="text" ng-model="url" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;">
```
우리는 `$http` service를 주입했고, input box에서 URL을 가져오고, POST request로 back-end에 전송했다. `success`와 `error` callback은 response를 다룬다. 
200 response일 경우에는 `success` handler가 되고, 아닐 경우에는 console에 response가 log로 남겨진다.

테스트하기 전에, back-end를 refactor하자. `/start`가 아직 없다.

# Refactor app.py

Redis job을 생성하는 것을 refactor하자.그리고 `get_counts()`라는 새로운 function을 추가하자:

```python
@app.route("/", methods=["GET", "POST"])
def index():
  return render_template('index.html')

@app.route("/starts", mehtods=["POST"])
def get_counts():
  data = json.loads(request.data.decode())
  url = data['url']
  if 'http://' not in url[:7]:
    url = 'http://'+url
    #start job
    job = q.enqueue_call(
      func=count_and_save_words, args=(url,), result_ttl=5000
    )
    #return created job id
    return job.get_id()
```
상단에 다음의 import를 하는 것을 확인해라:
```python
import json
```

이러한 수정사항은 간단할 것이다.

이제 테스트해보자. browser를 새로고침하고, 새로운 URL을 submit하자. 너는 javascript console에 job id를 확인 할 수 있을 것이다.
이제 angular는 polling Functionality에 추가할 수 있는 job id를 얻었다.


# Basic Polling

controller에 다음의 코드를 추가하여 *main.js*을 업데이트하자:
```javascript
function getWordCount(jobID){
  var timeout = "";
  
  var poller = function(){
    // fire anouter request
    $http.get('/results/'+jobID).
      success(function(data, status, headers, config){
        if(status === 202){
          $log.log(data, status);
        } else if(status === 200){
          $log.log(data);
          $timeout.cancel(timeout);
          return false;
        }
        // timeout이 취소되기 전까지
        // 매 2초마다 poller() function을 호출하게 한다.
        timeout = $timeout(poller, 2000);
      });
  };
  poller();
}
```

POST request의 success handler를 업데이트하자.
```javascript
$http.post('/start', {"url" : userInput})
  .success(function(results){
    $log.log(results);
    getWordCount(results);
  })
  .error(function(err){
    $log.log(err);
  });
```

controller에 `$timeout` servive를 inject하는 것을 잊지마라.

#### what's happening here?

1. 성공적인 HTTP request는 `getWordCount()` function을 호출하는 결과를 가져온다.
2. `poller()` function 내부에서는, `/results/job_id`를 호출한다.
3. `$timeout` service를 사용하면 word count를 가진 200 response가 돌아와 timeout이 취소될때 까지
매 2초마다 해당 function을 계속 호출할 수 있다.
`$timeout` service에 관한 내용은 [Angalar 문서](https://code.angularjs.org/1.4.9/docs/api/ng/service/$timeout)를 확인하라.

테스트를 할 때, Javascript console을 열어라. 다음과 같은 것들을 확인할 수 있다:
```javascript
Nay! 202
Nay! 202
Nay! 202
Nay! 202
Nay! 202
Nay! 202
Object {Course: 5, Download: 4, Python: 19, Real: 11, courses: 7…}
```

위 예제에서 `poller()` function은 7번 호출되었다. 처음 6번은 202을 return 했고, 
반면 마지막 호출에서는 word count object 를 가진 200 을 return 했다.

이제 DOM에 word count를 추가해야한다.

# Updating the DOM

*index.html*을 업데이트하자:
```html
<div class="container">
  <div class="row">
    <div class="col-sm-5 col-sm-offset-1">
      <h1>Wordcount 3000</h1>
      <br>
      <form role="form" ng-submit="getResults()">
        <div class="form-group">
          <input type="text" name="url" class="form-control" id="url-box" placeholder="Enter URL..." style="max-width: 300px;" ng-model="url" required>
        </div>
        <button type="submit" class="btn btn-default">Submit</button>
      </form>
    </div>
    <div class="col-sm-5 col-sm-offset-1">
      <h2>Frequencies</h2>
      <br>
      {% raw %}
      <div id="results">
        
          {{wordcounts}}
        
      </div>
      {% endraw %}
    </div>
  </div>
</div>
```
#### what did we change?

1. `input` tag가 `required` attribute를 가진다.
2. Jinja2 template 를 안쓸수 있다. Jinja2는 server side에서 제공되고, polling은 완전히 client side에서 다뤄지기 떄문에 우리는 Angular tag를 써야한다.
이 말은, Jinja2와 Angular 의 template tag 둘 다 `{{ }}`을 활용하기 때문에 우리는 `{% raw %}`와 `{% endraw %}`를 사용하여 Jinja2 tags를 벗어날 수 있다.
 더 많은 Angular tags를 사용해야 한다면, `$interpolateProvider`를 사용하여 AngularJS tag template을 사용하는 것이 좋은 방법이다. 더 많은 정보를 원한다면, [Angular docs](https://code.angularjs.org/1.4.9/docs/api/ng/provider/$interpolateProvider)를 확인하라.

두번째로, `poller()` function의 success handler를 업데이트 해라:
```javascript
success(function(data, status, headers, config){
  if(status === 202){
    $log.log(data, status);
  } else if(status === 200){
    $log.log(data);
    $scope.wordcounts = data;
    $timeout.cancel(timeout);
    return false;
  }
  timeout = $timeout(poller, 2000);
});
```
여기, `$scope` object에 results를 첨부했고 그래서 이것은 View에서 이용할 수 있다.

테스트하자. 모든게 잘 됐다면, DOM에서 object를 볼 수 있을 것이다. 예쁘지는 않지만, Bootstrap으로 쉽게 고정시켰고, `id=results`인 div 아래 다음의 코드를 추가해라.
그리고 위 코드에서 results div를 감쌌던 `{% raw %}`와 `{% endraw %}` tag를 제거해라:
```html
<div id="results">
  <table class="table table-striped">
    <thead>
      <tr>
        <th>Word</th>
        <th>Count</th>
      </tr>
    </thead>
    <tbody>
    {% raw %}
      <tr ng-repeat="(key, val) in wordcounts">
        
        <td>{{key}}</td>
        <td>{{val}}</td>
        
      </tr>
    {% endraw %}
    </tbody>
  </table>
</div>
```


[원문 : flask-by-example-part5](https://realpython.com/blog/python/flask-by-example-integrating-flask-and-angularjs/)