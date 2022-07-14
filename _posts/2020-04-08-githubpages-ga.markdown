---
layout:     post
title:      Github blog와 Google Analytics 연동하기 
author:     Soo-young Hwang
tags: 		Github-pages
# subtitle:  	
category:  study
---

### 서론
최근에 `Github pages`와 `jekyll`을 이용하여 블로그를 개설했습니다.    
네이버에서 블로그를 할 땐 유입 분석, 사용자 분석 등 다양한 통계 기능 서비스를 제공했었습니다.    
하지만 github pages는 이를 제공하지 않기에 Google analytics를 이용해야 합니다.    
몇 명이 이 글을 봤는지를 보는 것이 저한텐 동기부여가 되었기 때문에,       
저 또한, 제 **github blog에 Google Analytics(a.k.a GA)를 적용**하고자 합니다.   


### 본론
[analytics.goolgle.com](https://analytics.google.com/analytics/web/provision/#/provision) 접속하여 
`측정시작` 버튼을 누른 후, 계정을 만듭니다.   
사이트에서 요구하는 속성을 입력해 주고(속성이 생성됩니다.) 약관 동의를 확인합니다.    
그럼 관리자 > 속성 페이지가 뜨며 추적 ID와 상태 등이 나와있는 것을 볼 수 있습니다.

이 추적 ID를 복사하여 
jekyll project repository 루트 디렉토리에 있는 _config.yml을 수정합니다

/_config.yml
{% highlight bash %}
g-analytics: 추적 ID
{% endhighlight %}

다시 GA사이트로 들어가 잘 적용이 되었는지 확인하면 끝입니다.   
간단

### 결론
![Description](http://swimminghwang.github.io/img/githubpages-ga1.png)

### references
[inasie.github.io](https://inasie.github.io/it%EC%9D%BC%EB%B0%98/1/)