---
layout: post
title:  "part8"
date:   2017-02-28 21:08:53 +0900
categories: translation
---

이 포스팅은 [원문](https://realpython.com/blog/python/flask-by-example-custom-angular-directive-with-D3/)을 의역하고 필요한 부분은 추가한 것.


> Part eight : Javascript와 D3를 사용하여 빈도 분포를 표시하기 위해 custom Angular Directive를 생성하자

# Current User Interface

terminal 창에서 Redis server를 실행하자:
```python
$ redis-server
```

그 다음 다른 창에서 worker를 실행하자:
```python
$ cd flask-by-example
$ python worker.py
17:11:39 RQ worker started, version 0.4.6
17:11:39
17:11:39 *** Listening on default...
```

마지막으로, 세번째 창에 app을 실행하라:
```python
$ cd flask-by-example
$ python manage.py runserver
```

word count가 작동되는 것을 확인할 수 있다. 이제 결과를 D3로 나타내기 위해 custom Angular Directive를 추가할 수 있다.

# Angular Directive

*index.html* 파일에 D3 library를 추가하자:
```html
<!-- scripts -->
<script src="//d3js.org/d3.v3.min.js" charset="utf-8"></script>
<script src="//code.jquery.com/jquery-2.2.1.min.js"></script>
<script src="//netdna.bootstrapcdn.com/bootstrap/3.3.6/js/bootstrap.min.js"></script>
<script src="//ajax.googleapis.com/ajax/libs/angularjs/1.4.5/angular.min.js"></script>
<script src=""></script>
```

이제 새로운 custom directive를 설정하자.

[Angular Directive](https://code.angularjs.org/1.4.5/docs/guide/directive)는 
특별한 event와 attribute를 가진 HTML section을 삽입할 수 있게 해주는 DOM element의 marker이다.
*main.js*의 controller 아래에 다음의 코드를 추가하여 Directive의 첫 부분을 설계하자.
```javascript 
.directive('wordCountChart', ['$pars', function($parse){
    return {
        restirct : 'E',
        replace : true,
        template : '<div id="chart"></div>',
        link : function(scope) {}
    };
}]);
```

`restrict:'E'`는 Directive를 HTML element로 제한한되는 Directive로 생성한다.
`replace:true`는 `template`의 HTML Directive로 간단하게 대체한다. 
`link` function은 controller에 정의된 scope의 변수에 접근할 수 있게 한다.

그 다음에, 변수의 변화를 "watch"하고 적절하게 response하기 위해 `watch` function을 추가하라.  
다음과 같이 `link` function을 추가하자:
```javascript
link : function(scope){
    scope.$watch('wordcounts', function(){
        // 여기에 코드 추가
    }, true);
}
```

마지막으로, `<div class="row">`의 근처의 divider 아래에 Directive를 추가하라.:
```html
<br>
<word-count-chart data="wordcounts"></word-count-chart>
```

Directive 설정과 D3 libraray에 주목해라...

# D3 Bar Chart

D3는 DOM에서 data를 보여주기 위해 HTML, CSS, SVG를, interactive하게 만들기 위해 Javascript를 활용하는 강력한 library이다.  
기본적인 bar chart를 만드는 데 사용해보자:
```javascript
scope.$watch('wordcounts', function(){
    var data = scope.wordcounts;
    for(var word in data){
        d3.select('#chart')
            .append('div')
            .selectAll('div')
            .data(word[0])
            .enter()
            .append('div');
    }
}, true);
```
이제, 언제든지 `wordcounts`가 변경되면 이 function은 호출되고, DOM이 업데이트 된다. 
object가 AJAX request로 return 되기 떄문에 , 우리는 chart에 iterator로 특정 data를 추가할 수 있다.
특별히, 모든 word는 [data join](https://bost.ocks.org/mike/join/)을 거쳐 새로운 `div`에 추가된다. 

이 코드를 실행시켜봐라.

무슨 일이 일어나나? 아무것도 일어나지 않는다. 맞나? 새로운 site를 submit하고 난 뒤, Chrome의 Developer Tools를 확이해봐라 
많은 nested `div`를 볼 수 있다. style을 추가할 필요가 있다...

# step 2: Styling the Bar Chart

간단한 CSS로 시작하자:
```css 
#chart {
  overflow-y: scroll;
}

#chart {
  background: #eee;
  padding: 3px;
}

#chart div {
  width: 0;
  transition: all 1s ease-out;
  -moz-transition: all 1s ease-out;
  -webkit-transition: all 1s ease-out;
}

#chart div {
  height: 30px;
  font: 15px;
  background-color: #006dcc;
  text-align: right;
  padding: 3px;
  color: white;
  box-shadow: 2px 2px 2px gray;
}
```

HTML page에 Bootstrap stylesheet 아래에 다음을 추가하라:
```html
<link rel="stylesheet" type="text/css" href="../static/main.css">
```

이제 당신이 website를 검색하면, 왼쪽부터 얇은 파란색 bar가 있는 회색 구역을 볼 수 있다.

전체 중의 10개의 data의 각각 생성된 bar를 볼 수 있다. 
그러나, 가독성을 위해 각 bar의 너비를 증가시키기 위해서는 D3 code를 수정해야한다.

# Step 3: Making the Bar Chart more Interactive 

[D3 style function](https://github.com/d3/d3-3.x-api-reference/blob/master/Selections.md#style)을 channing방식으로 추가할 수 있다.

```javascript
scope.$watch('wordcounts', function() {
  var data = scope.wordcounts;
  for (var word in data) {
    d3.select('#chart')
      .append('div')
      .selectAll('div')
      .data(word[0])
      .enter()
      .append('div')
      .style('width', function() {
        return (data[word] * 20) + 'px';
      })
      .text(function(d){
        return word;
      });
  }
}, true);
```

이제 우리는 홈페이지에 나온 단어 수에 따라 동적으로 너비를 생성할 수 있다.

```javascript
.style('width', function() {
  return (data[word] * 20) + 'px';
})
.text(function(d){
  return word;
});
```

이 style은 각 단어의 갯수에 20을 곱하고 'pixel'로 변환하여 return 하게 계산한다.
또한, 각 bar element에 출현한 단어를 text로 추가할 수 있다.

여전히 한 개가 부족하다. 새로운 website를 찾은 동안은 어떻게 되나? 시도해 봐라.
새로운 char는 예전 것 아래에 추가된다. 우리는 새로운 것을 생성하기 전에 전의 것을 초기화 할 필요가 있다.

# Step 4: Clean up for the Next URL Search

Directive의 `link` function을 업데이트 하자:
```javascript
link: function (scope) {
  scope.$watch('wordcounts', function() {
    d3.select('#chart').selectAll('*').remove();
    var data = scope.wordcounts;
    for (var word in data) {
      d3.select('#chart')
        .append('div')
        .selectAll('div')
        .data(word[0])
        .enter()
        .append('div')
        .style('width', function() {
          return (data[word] * 20) + 'px';
        })
        .text(function(d){
          return word;
        });
    }
  }, true);
}
```

`d3.select('#chart').selectAll('*').remove();`는 `scope.$watch` function이 실행될 때마다 간단하게 chart를 초기화한다.
이제 우리는 새롭게 사용할 때마다 깨끗한 chart를 얻을 수 있다.

이제 Application을 완성했다!!!




[원문 : flask-by-example-part6](https://realpython.com/blog/python/flask-by-example-custom-angular-directive-with-D3/)