---
title: Selenium으로 위키피디아 표 크롤링하기
author: jisu
date: 2025-05-05 11:33:00 +0900
categories: [Crawling]
tags: [Selenium]
pin: false
math: true
mermaid: true
comments: true
---

## Wikipedia 크롤링, 왜 requests는 안 되고 Selenium은 될까?
Wikipedia에서 공항 IATA 코드 표 데이터를 크롤링하려 했다. 처음에는 당연히 requests와 BeautifulSoup을 이용해 간단히 HTML에서 표를 추출하면 될 것이라고 생각했다.
실제로 페이지를 열어보면 정적 테이블 구조가 눈에 띄기 때문이다.
하지만 예상과 다르게, requests로 페이지를 요청하고 .find("table")을 사용해도 테이블이 전혀 나타나지 않았다.
처음엔 개발자도구를 잘못 확인했나 싶어서 여러 번 확인해봤지만 잘못된 건 없었다.

### 🚨 문제 원인 : User-Agent
Wikipedia는 자동화된 비정상 요청을 감지할 경우, 일반 브라우저가 아닌 요청에 대해서는 축약된 HTML을 내려주는 경우가 있다.
즉, requests는 사람처럼 보이지 않기 때문에 Wikipedia가 완전한 문서를 제공하지 않았던 것이다.

일반적인 requests 요청은 다음과 같이 브라우저 정보가 생략되어 있다.

res = requests.get(url)

이를 방지하기 위해 User-Agent를 브라우저처럼 설정했다.

```python
headers = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/123.0.0.0 Safari/537.36"
    )
}
res = requests.get(url, headers=headers)
```

하지만 이렇게 해도 문제는 해결되지 않았다. Wikipedia는 여전히 비브라우저 요청을 차단하고 있었다.

#### User-Agent란?
**User-Agent**는 브라우저나 프로그램이 웹 서버에 요청을 보낼 때 자신의 정체를 밝히는 헤더다.  
웹 서버는 이 정보를 통해 요청이 크롬 브라우저에서 온 것인지, 모바일인지, 아니면 자동화된 봇인지 판단한다.  
많은 웹사이트는 봇처럼 보이는 요청에 대해 데이터를 축소하거나 차단하기 때문에, 크롤링할 때는 이 User-Agent 설정이 매우 중요하다.

### 💡 Selenium으로 전환
결국 Selenium을 사용해 브라우저 자체를 띄우고 페이지를 로딩한 후, BeautifulSoup으로 page_source를 파싱하는 방식으로 전환했다.
이 경우 실제 브라우저가 요청을 보내기 때문에 Wikipedia는 전체 HTML을 정상적으로 내려주었고, 테이블도 정확히 추출할 수 있었다.
Selenium을 사용할 때는 User-Agent를 신경 쓰지 않아도 되었다.
실제 브라우저가 열리는 것이기 때문에, 서버 입장에서는 사람이 접속한 것과 동일하게 처리하기 때문이다.

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from bs4 import BeautifulSoup
import pandas as pd
import time

# A~Z 대상 알파벳 리스트
alphabet_list = [chr(i) for i in range(65, 91)]
data = []

# Chrome 옵션 설정
options = Options()
options.add_argument("--headless")  # 창 없이 실행
options.add_argument("--disable-gpu")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")

# 크롬 드라이버 실행
driver = webdriver.Chrome(options=options)

for letter in alphabet_list:
    url = f"https://en.wikipedia.org/wiki/List_of_airports_by_IATA_airport_code:_{letter}"
    
    try:
        driver.get(url)
        time.sleep(2)  # 페이지 로딩 대기

        soup = BeautifulSoup(driver.page_source, "html.parser")
        table = soup.find("table", class_="wikitable sortable jquery-tablesorter")

        if not table:
            print(f"테이블 없음: {url}")
            continue

        rows = table.select("tbody > tr")
        for row in rows:
            cols = row.find_all("td")
            if len(cols) >= 4:
                iata = cols[0].text.strip()
                airport_name = cols[2].get_text(strip=True)
                location = " ".join(cols[3].stripped_strings)
                data.append([iata, airport_name, location])

    except Exception as e:
        print(f"오류 발생: {e}")

driver.quit()  # 드라이버 종료

# DataFrame 저장
df = pd.DataFrame(data, columns=["IATA", "Airport Name", "Location"])
df.to_csv("airports.csv", index=False)
print("전체 크롤링 완료")
```
이렇게 하면 csv파일이 잘 저장됨을 확인할 수 있다.

