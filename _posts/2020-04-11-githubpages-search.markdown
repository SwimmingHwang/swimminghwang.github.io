---
layout:     post
title:      github blog를 검색엔진에서 검색되도록 설정하기
author:     Soo-young Hwang
tags: 		Github-pages
# subtitle:  	
category:  study
---

### 서론
[지난 포스트](https://swimminghwang.github.io/study/2020/04/08/githubpages-ga/)에서 내 블로그의 방문자 수 혹은 방문자 유입을 분석해주는 `Google Analytics`를 적용해 보았습니다.    
`Github-pages`를 이용한 블로그는 포털사이트에 검색해도 안 나오더라구요...    
따라서 검색 가능하게 등록을 직접 해 줘야 한답니다 ^ㅗ^    
이번 포스트에선 **포털사이트에 검색했을 때 내 블로그가 나올 수 있도록 등록하는 과정**을 다뤄보았습니다.     
아래의 참고 블로그와 같이, 대표적인 포털 사이트 Google, Daum, Naver 와 같은 검색엔진에 등록해 보았습니다.

### 본론

검색엔진에 등록하기에 앞서 검색 엔진이 웹사이트를 읽어가는 작업을 정상적으로 진행하기 위해서는 `sitemap.xml`(구글)과 `feed.xml`(네이버, 다음) 그리고  `robots.txt` 이 필요합니다.    
따라서 두 파일을 작성하고 검색엔진에 등록하는 순서로 진행됩니다.

1. sitemap.xml, feed.xml 작성
2. robotx.txt 만들기
3. 포털사이트에 등록하기

`sitemap.xml` : 웹사이트 내 모든 페이지의 목록을 나열한 파일로 책의 목차와 같은 역할<br/>`robots.txt` : 검색 엔진 크롤러에서 사이트에 요청할 수 있거나 요청할 수 없는 페이지 설정하는 부분 및 제어하는 부분
검색 로봇들에게 웹사이트의 사이트맵이 어디 있는지 알려주는 역할  

<blockquote>Step 1. 사이트맵(sitemap), RSS feed 작성</blockquote>

#### 사이트맵 작성

**사이트맵은 페이지 정보를 포털사이트에 알리는 역할**을 합니다.        
그러므로 사이트맵을 만들어서 Google에 등록을 해두면 Google에서 주기적으로 Web상에 있는 contents를 수집합니다.   즉, 제 블로그를 등록하면 제 블로그를 수집합니다.   그럼 구글에 관련 검색어로 색인이 되고, 구글링시 제 글을 찾아볼 수 있겠죠!?! 야호 신난당   
참고 블로그에서 Github-pages에서는 보안상의 문제로 플러그인을 사용할 수가 없어서 jekyll plugin을 사용하지 않고 sitemap.xml을 만들어야 한다고 합니다. 사이트를 제너레이트하고 깃헙에 푸시하면 대부분 문제없이 사용할 수 있다고 하네요    

<p><execode>Todo : 루트 디렉터리에 `sitemap.xml` 을 생성한 후, 아래의 코드를 삽입</execode></p>   
- 아래 코드에 있는 site.url 변수가 사용될 수 있도록 `.config.yml`에 url 변수를 꼭 넣어주기    
- 주소 URL에 가능한 영문으로 만들고, 특수기호(&,%..)는 사용하지 않기

{% highlight yml %}
url: "https://swimminghwang.github.io"
{% endhighlight %}


{% highlight xml %}
{% raw %}
---   
--- //머리말 필수!   
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.sitemaps.org/schemas/sitemap/0.9 http://www.sitemaps.org/schemas/sitemap/0.9/sitemap.xsd" xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {% for post in site.posts %}
    <url>   
      <loc>{{ site.url }}{{ post.url }}</loc>
      {% if post.lastmod == null %}
        <lastmod>{{ post.date | date_to_xmlschema }}</lastmod>
      {% else %}
        <lastmod>{{ post.lastmod | date_to_xmlschema }}</lastmod>
      {% endif %}
      {% if post.sitemap.changefreq == null %}
        <changefreq>weekly</changefreq>
      {% else %}
        <changefreq>{{ post.sitemap.changefreq }}</changefreq>
      {% endif %}
      {% if post.sitemap.priority == null %}
          <priority>0.5</priority>
      {% else %}
        <priority>{{ post.sitemap.priority }}</priority>
      {% endif %}
    </url>
  {% endfor %}
</urlset>
{% endraw %}
{% endhighlight %}


`블로그주소/sitemap.xml`을 입력하면 아래의 이미지처럼 본인의 블로그 사이트 맵을 볼 수 있습니다.    

![img](http://swimminghwang.github.io/img/sitemap.png){: width="600" height="400"}

#### Rss feed 작성

<p><execode>Todo : 루트 디렉터리에 `feed.xml` 을 생성한 후, 아래의 코드를 삽입</execode></p> 
{% highlight xml%}
{% raw %}
---
layout: null
---
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{{ site.title | xml_escape }}</title>
    <description>{{ site.description | xml_escape }}</description>
    <link>{{ site.url }}{{ site.baseurl }}/</link>
    <atom:link href="{{ "/feed.xml" | prepend: site.baseurl | prepend: site.url }}" rel="self" type="application/rss+xml"/>
    <pubDate>{{ site.time | date_to_rfc822 }}</pubDate>
    <lastBuildDate>{{ site.time | date_to_rfc822 }}</lastBuildDate>
    <generator>Jekyll v{{ jekyll.version }}</generator>
    {% for post in site.posts limit:30 %}
      <item>
        <title>{{ post.title | xml_escape }}</title>
        <description>{{ post.content | xml_escape }}</description>
        <pubDate>{{ post.date | date_to_rfc822 }}</pubDate>
        <link>{{ post.url | prepend: site.baseurl | prepend: site.url }}</link>
        <guid isPermaLink="true">{{ post.url | prepend: site.baseurl | prepend: site.url }}</guid>
        {% for tag in post.tags %}
        <category>{{ tag | xml_escape }}</category>
        {% endfor %}
        {% for cat in post.categories %}
        <category>{{ cat | xml_escape }}</category>
        {% endfor %}
      </item>
    {% endfor %}
  </channel>
</rss>
{% endraw %}
{% endhighlight %}

<blockquote>Step 2. robots.txt 만들기 </blockquote>

초반에 설명했듯, `robots.txt` 파일에 `sitemap.xml` 파일의 위치를 등록해 두면 검색엔진의 크롤러들이 크롤링을 하든데 있어서 도움을 준다. 
<p><execode>Todo : 루트 디렉터리에 `robots.txt` 을 생성한 후, 아래의 코드를 삽입</execode></p> 
{% highlight xml %}
<!--허용할 검색엔진 명
User-agent : Googlebot   
User-agent : NaverBot
-->
User-agent: *    
Allow: / 
<!-- 크롤링이 되지 않았으면 하는 페이지가 있다면
Disallow: /pagename/   
-->
Sitemap: https://본인의 블로그 주소/sitemap.xml
{% endhighlight %}

<blockquote>Step 3. 포털사이트에 등록하기</blockquote>

**1. 구글**   
[Google Search Console](https://search.google.com/search-console/about?hl=ko&utm_source=wmx&utm_medium=wmx-welcome)
도메인 말고 URL 접두어를 선택하여 블로그 주소를 입력하면 됩니다.   
Google 애널리틱스를 적용해두었더니 소유권이 자동으로 확인되었습니다.   

![gs2](https://swimmingHwang.github.io/img/gs2.png){: width="300" height="300"}   
적용하지 않았다면 [Google 애널리틱스 적용하기](https://swimminghwang.github.io/study/2020/04/08/githubpages-ga/) 를 참고하여 적용하거나 [참고 블로그](https://honbabzone.com/jekyll/start-gitHubBlog/#step-9-%EA%B5%AC%EA%B8%80-%EA%B2%80%EC%83%89-%EA%B0%80%EB%8A%A5%ED%95%98%EA%B2%8C-%ED%95%98%EA%B8%B0) 를 통해 소유권 확인 과정을 거치시면 됩니다.     
다시 [Google Search Console](https://search.google.com/search-console/about?hl=ko&utm_source=wmx&utm_medium=wmx-welcome) 에 접속하여 좌측에 Sitemaps 버튼을 눌러 `sitemap.xml`을 입력하여 등록하면 됩니다.    

![gsc](https://swimmingHwang.github.io/img/gsc.png)


**2. Naver**   
[Naver Search Advisor](https://searchadvisor.naver.com/) 들어가서 로그인 후 오른쪽 상단에 **웹마스터 도구** 로 들어가서 사이트 등록을 진행합니다.   
소유권 인증을 받기 위해 html파일을 다운받고 루트디렉터리에 업로드하여 인증을 받습니다.   
그다음 좌측 요청 -> RSS 제출을 눌러 앞서 작성한 `feed.xml` 을 제출합니다.   

**3. Daum**   
[Daum 검색 등록](https://register.search.daum.net/index.daum) 들어가서 url을 등록합니다.   
(RSS가 있었다고 하는데 안 보여요ㅜㅜ)



### 결론
이로써 블로그 시작 준비를 한 ..70% 완료한 것 같습니댜햐햐 
남은 할일 : 
1. 카테고리랑 태그 구분하기
2. 본문 속 title css 최적화하기
3. disqus 잘 되는지 확인하기 
4. 포털사이트 검색 잘 되는지 확인하기 
5. **공부한 후 포스트 성실히 올리기**

깨닳은 점 :
너무 꼼꼼히 하려다가 일주일걸린다 대충대충 일단 하자~~ 

블로그 포스트랑 같이 작성하다가 이틀이나 걸렸다..   
그만두고싶다.....
하지만 포털사이트 등록까지 해내따아ㅏㅏ 블로그 시작준비를 마쳤다 ㅠㅠ   
조금 뿌듯하다 ~,~ 

### references
[참고 사이트1](http://jinyongjeong.github.io/2017/01/13/blog_make_searched/)   
[참고 사이트2](http://dveamer.github.io/homepage/Sitemap.html)   
[참고 사이트3](https://honbabzone.com/jekyll/start-gitHubBlog/#step-9-%EA%B5%AC%EA%B8%80-%EA%B2%80%EC%83%89-%EA%B0%80%EB%8A%A5%ED%95%98%EA%B2%8C-%ED%95%98%EA%B8%B0)     
[참고 사이트4](https://wayhome25.github.io/etc/2017/02/20/google-search-sitemap-jekyll/)

