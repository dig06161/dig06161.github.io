---
published: true
image: /img
layout: post
title: Python Http POST요청 자동화 코드 
tags: [python, POST, http, 자동화, 웹 요청, 웹 자동화]
math: true
date: 2022-01-25 20:13
---

<hr>

## 파이썬 POST 자동화 코드  

<br/>

ISMS-P(정보보호 및 개인정보보호관리체계)인증을 충족하기 위해 웹 모의 해킹을 진행했던 경험이 있습니다. 이 ISMS-P 검사 항목중에 자동화 공격에 대한 진단 항목이 있습니다. 이 항목을 진단할때 python코드를 작성해 반복문으로 POST요청을 다량으로 발생시켜 공격을 시도 했습니다.

이에 관련된 코드를 올려봅니다.

단순히 while문으로 반복적인 요청을 보내는 코드입니다.

```python
import urllib.parse
import urllib.request
a = 1

while(a <= 100):
    #웹 POST요청에 대한 파라미터들을 json형식으로 정의해 줍니다.
    details = urllib.parse.urlencode({'mm':'voc', 'sm':'ins', 'pg':'1', 'regCd':'', 'regName':'',
                                     'Title':'취약점 점검중', 'WriterName':'yunjoker',
                                     'Tel1':'123', 'Tel2':'123', 'Tel3':'123', 'EmailId':'123',
                                     'EmailDomain':'naver.com', 'agree1':'1', 'Content':'ㅁㄴㅇㄹ'})
    #인코딩 유형을 설정합니다
    details = details.encode('UTF-8')

    #요청 URL을 추가합니다
    url = urllib.request.Request("https://***/common/process/process.inquiry.php", details)
    
    #Burp Suite 등의 프로그램이나 기능을 이용해 쿠키값들을 그대로 헤더에 추가해 줍니다.
    url.add_header("POST", "/common/process/process.inquiry.php HTTP/1.1")
    url.add_header("Host", "***")
    url.add_header("Connection", "close")
    url.add_header("Content-Length", "360")
    url.add_header("Accept", "application/json, text/javascript, */*; q=0.01")
    url.add_header("X-Requested-With", "XMLHttpRequest")
    url.add_header("User-Agent",
                   "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.120 Safari/537.36")
    url.add_header("Sec-Fetch-Mode", "cors")
    url.add_header("Content-Type", "application/x-www-form-urlencoded")
    url.add_header("Sec-Fetch-Site", "same-origin")
    url.add_header("Referer", "***")
    url.add_header("Accept-Encoding", "gzip, deflate")
    url.add_header("Accept-Language", "ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7")
    url.add_header("Cookie", "_ga=GA1.2.1167119943.1572314702; _gid=GA1.2.304601219.1572314702; PHPSESSID=n07lta0bu412bcsk0q9jejvoh7; _gat_gtag_UA_74704901_2=1")
    
    #POST요청 후 리스폰스 데이터를 ResponseData 변수에 저장합니다.
    ResponseData = urllib.request.urlopen(url).read().decode("utf-8")
    
    #리스폰스 데이터를 출력합니다
    print(ResponseData)
    
    a = a+1
```


<hr>