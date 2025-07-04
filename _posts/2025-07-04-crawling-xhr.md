---
title: 웹 크롤링 - DevTools로 구조 파악하기
author: jisu
date: 2025-07-04 14:00:00 +0900
categories: [Crawling]
tags: [DevTools, XHR]
pin: false
math: false
mermaid: false
comments: true
---

데이터를 크롤링했을때, 사이트에 들어가 보니 ‘더보기’ 버튼이 있어서,
당연히 **Selenium**으로 버튼을 누르며 데이터를 수집해야겠다고 판단했다.
그러나 예상처럼 동작하지도 않았다.

---

## 🔍 DevTools로 네트워크 요청 확인하기

크롬의 **개발자도구 (DevTools)** 를 열고 `Network` 탭으로 이동했다.

1. 필터 중 **`XHR` 또는 `Fetch`** 를 선택  
   → 이 필터는 JavaScript를 통해 백엔드 서버와 데이터를 주고받는 요청만 보여준다.
2. 다시 ‘더보기’ 버튼을 눌러보았다.


클릭과 동시에 다음과 같은 **POST 요청**이 전송되고 있었다:

```
POST https://dayforyou.com/searchContent
```
이 API는 특정 날짜, 카테고리, 페이지 번호를 기준으로 팝업스토어 목록 HTML을 반환하는 역할을 한다.
즉, 페이지 하단의 '더보기' 버튼은 이 API에 `page=2`, `page=3` 식으로 반복 요청을 보내며 리스트를 불러오는 구조였다.

Payload 탭을 확인해보면 아래와 같은 파라미터가 함께 전송된 걸 확인할 수 있었다.

![image](https://github.com/user-attachments/assets/e7c13b02-0841-435e-a10b-37a45efe44c8)

```json
{
  "searchDate": "2025-07-04",
  "searchCategory": "food",
  "page": 2
}
```

'더보기' 버튼은 실제로 DOM을 조작하거나, JavaScript 이벤트를 직접 트리거하는 구조가 아니었고
단순히 이 searchContent API에 파라미터만 바꿔가며 요청을 보내는 식이었다.

## XHR (XMLHttpRequest)이란? 
XHR은 **웹페이지가 새로고침 없이 서버와 데이터를 주고받기 위한 방식**이다.
대부분의 웹사이트는 페이지 전체를 새로고치기보단,  
**특정 영역에 필요한 데이터만 가져오는 비동기 방식**으로 구성돼 있다.

주로, 이런 경우에 XHR 요청이 발생한다:
- 로그인 시, 아이디/비밀번호 전송
- 댓글 등록/삭제 시 실시간 반영
- 스크롤 내릴 때 자동으로 콘텐츠 추가 로딩
- API 요청으로 유저 정보 가져오기

## 💡 해결: requests로 간단하게 크롤링
Selenium이 아니라 `requests.post()`와 `BeautifulSoup`만으로도 충분했다.

```python
url = "https://dayforyou.com/searchContent"
data = {
    "searchDate": "2025-07-04",
    "searchCategory": "food",
    "page": 1
}
res = requests.post(url, data=data)
```

페이지 파라미터만 바꾸면 다음 목록도 받아올 수 있고,
팝업 정보를 짧은 시간 안에 안정적으로 수집할 수 있었다.

## 📌 언제 DevTools를 꼭 봐야 할까?
크롤링을 할 때 `DevTools > Network` 탭을 보면 이런 걸 확인할 수 있다:

- 어떤 API가 호출되는지 (`XHR`, `Fetch` 필터 사용)

- 어떤 파라미터와 메서드 (`GET`, `POST`)가 전송되는지

- 어떤 데이터 구조 (`JSON`, `HTML`, 등)가 응답되는지


이걸 먼저 보면, `Selenium` 없이 `requests` 또는 API 기반 수집이 가능한지 판단 가능하다.


## 마무리
DevTools를 먼저 열어 네트워크 구조를 확인해보면,
불필요한 도구 사용 없이 훨씬 빠르고 안정적인 방식을 선택할 수 있다.

