---
layout: post
title:  "part6"
date:   2017-02-28 19:41:42 +0900
categories: translation
---

이 포스팅은 [원문](https://realpython.com/blog/python/flask-by-example-updating-the-ui/)을 의역하고 필요한 부분은 추가한 것.


> Part seven : 더 user-friendly하도록 front-end를 update하자.

# Current User Interface

terminal 창에 Redis server를 시작하자
```python
$ redis-server
```

그리고 다른 창에 worker를 실행하라:
```python
$ cd flask-by-example
$ python worker.py 
17:11:39 RQ worker started, version 0.5.6
17:11:39
17:11:39 *** Listening on default...
```

마지막으로, 세번째 창에 app을 실행하라.
```python
$ python manage.py runserver
```

제대로 작동하는지 테스트하보자

몇개를 수정하자:
1. submit한 site를 count하는 동안 반복적이 click을 막기 위해 비활성된 submit button으로 시작할 것이다.
2. 그 다음, application이 word를 count하는 동안 loading spinner를 보여줄 것이다. 
3. 마지막으로, domain이 유효하지 않으면 error를 표시해 줄 것이다.

# Changing the button 

HTML의 button을 수정하자:
```html 
{% raw %}
    <button type="submit" class="btn btn-primary" ng-disabled="loading">{{submitButtonText}}</button>
{% endraw %}
```

`loading`을 첨부한 `ng-disabled` directive를 추가했다. 이것은 `loading` 값이 `true`일 때 button을 비활성화시킨다.
그 다음으로, view에 표시하기 위해 `submitButtonText`라는 변수를 추가한다.이 방법으로 하면 우리는 text를 `loading...`과 `Submit`으로 바꿀 수 있어 사용자가 지금 어떻게 되고 있는 지 알 수 있다.
그리고 우리는 button을 `{% raw %}`와 `{% endraw %}`로 감싸서 Jinja가 이것을 그냥 HTML이라고 생각하게 한다. 
만약 이것을 하지 않는다면, Flask는 `{{submitButtonText}}`를 Jinja 변수로 처리하려고 할 것이며 Angular는 처리할 기회조차 없을 것이다.

동반된 Javascript는 굉장히 간단하다.

*main.js*의 `WordCountController`의 맨 위 부분에서 다음의 코드를 추가하라:
```javascript 
$scope.submitButtonText = 'Submit';
$scope.loading = false;
```

`loading`의 값을 `false`로 초기화해서 button이 비활성화되지 않을 것이다. 또한 button의 text를 `Submit`으로 초기화한다.

POST call을 다음과 같이 변경하라:
```javascript
$http.post("/start", {'url' : userInput})
    .success(function(results){
        $log.log(results);
        getWordCount(results);
        $scope.wordcounts = null;
        $scope.loading = true;
        $scope.submitButtonText = "Loading...";
    })
    .error(function(error){
        $log.log(error);
    });
```

3 줄을 추가했다.
1. `wordcounts`를 `null`로 설정하여 오래된 값은 제거했다.
2. `loading`을 `true`로 해서 `ng-disabled` directive를 통해 loading button은 비활성화 될 것이다.
3. `submitButtonText`를 `Loading...`으로 설정해서 사용자가 button이 비활성화 된 것을 알게 한다.

다음은 `poller` function을 업데이트 하자:
```javascript
var poller = function(){
    $http.get('/results/'+jobID)
        .success(function(data, status, headers, config){
            if(status === 202){
                $log.log(data, status);
            }else if(status === 200){
                $log.log(data);
                $scope.loading = false;
                $scope.submitButtonText = "Submit";
                $scope.wordcounts = data;
                $timeout.cancel(timeout);
                return false;
            }
            timeout = $timeout(poller, 2000);
        });
};
```
result가 성공적일 때, `loading`을 다시 `false`로 변경하여 button을 다시 활성화시키고 
사용자에게 다시 새로운 URL을 submit할 수 있다는 것을 알리기 위해 button의 text를 `Submit`을 변경한다.

# Adding Spinner 

다음으로 wordcount section 아래에 spinner를 추가하여 사용자가 지금 무슨 일이 일어나는지 알게 하자. 
이것은 아래에서 보이는 것와 같이 results `div`아래에 움직이는 gif를 추가함으로서 실행된다.

```html
<div class="col-sm-5 col-sm-offset-1">
  <h2>Frequencies</h2>
  <br>
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
  <img class="col-sm-3 col-sm-offset-4" src="{{ url_for('static',
  filename='spinner.gif') }}" ng-show="loading">
</div>
```

[이곳](https://github.com/realpython/flask-by-example/tree/master/static)에서 *spinner.gif*를 다운받아라.
너는 button과 같이, `loading`이 첨가된 `ng-show`를 볼 수 있다. 이 방법은 `loading`이 `true`일 때 spinner gif가 보일 것이고,
`loading`이 `false`일 때 spinner는 사라질 것이다.

# Dealing with errors

마지막으로, 사용자가 유효하지 않은 URL을 submit했을 경우를 다루길 원한다. 
form 아래에 다음의 코드를 추가하라:
```html
<div class="alert alert-danger" role="alert" ng-show='urlerror'>
  <span class="glyphicon glyphicon-exclamation-sign" aria-hidden="true"></span>
  <span class="sr-only">Error:</span>
  <span>There was an error submitting your URL.<br>
  Please check to make sure it is valid before trying again.</span>
</div>
```
이것은 사용자가 나쁜 URL을 submit했을 경우, warning dialog를 보여주기 위해 bootstrap의 `alert` class를 사용했다. 
우리는 `urlerror`가 `true`일 경우에만 dialog를 보여주기 위해 Angular의 `ng-show` directive를 사용했다.

마지막으로, `WordCountController`에서 경고를 초기에 보여주지 않기 위해 `$scope.urlerror`를 `false`로 초기화했다. 

```javascript
$scope.urlerror = false;
```

`poller` function에서 errors를 catch하자:
```javascript
var poller = function(){
    $http.get("/start"+jobID)
        .success(function(data, status, headers, config){
            if(status===202){
                $log.log(data);
            }else if(status === 200){
                $log.log(data);
                $scope.loading= false;
                $scope.submitButtonText = "Submit";
                $scope.wordcounts = data;
                $timeout.cancel(timeout);
                return false;
            }
            timeout = $timeout(poller, 2000);
        })
        .error(function(err){
            $log.log(err);
            $scope.loading = false;
            $scope.submitButtonText = "Submit";
            $scope.urlerror = true;
        });
};
```

이것은 console에 error를 기록하고, `loading`을 `false`로 변경하고, submit button의 text를 다시 `Submit`으로 변경하라.
그러면 다시 submit할 수 있게 된고, `urlerror`를 `true`로 바꾸면, waring이 보일 것이다.

마지막으로, `/start`의 POST call의 `success` function에서 `urlerror`를 `false`로 설정하라.

```javascript
$scope.urlerror = false;
```

이제 사용자가 새로운 url을 submit할 때 warning은 사라질 것이다.

지금까지 user interface를 좀 깔끔하게 정리했고, 화면 뒤에서 word count 기능이 실행되는 동안 사용자는 무슨 일이 일어나고 있는지 알 수 있게 되었다.



[원문 : flask-by-example-part6](https://realpython.com/blog/python/flask-by-example-updating-the-ui/)