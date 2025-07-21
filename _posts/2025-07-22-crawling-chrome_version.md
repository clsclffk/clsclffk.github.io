---
title: Chrome 자동 업데이트와 드라이버 버전 불일치
author: jisu
date: 2025-07-22 00:00:00 +0900
categories: [Crawling]
tags: [Selenium, ChromeDriver, Crawling]
pin: false
math: false
mermaid: false
comments: true
---

며칠 전까지 정상적으로 작동하던 `Selenium` 기반의 크롤러가  
갑자기 `NoSuchDriverException`과 함께 실행되지 않기 시작했다.

```
selenium.common.exceptions.NoSuchDriverException: Message: Unable to obtain driver for chrome
```

다음 명령어로 버전을 확인해보니,

```bash
google-chrome --version
# Google Chrome 137.0.7151.103

chromedriver --version
# ChromeDriver 120.0.6099.109
```

Chrome과 드라이버 버전이 불일치하고 있었다.

## 🔍 원인
Selenium은 크롬 브라우저와 ChromeDriver의 버전이 정확히 일치해야 정상적으로 동작한다.
그러나 EC2 서버에서는 `apt` 패키지 관리 시스템이 자동으로 크롬을 최신 버전으로 업데이트하고 있었다.

이는 APT의 자동 업데이트 설정이 활성화된 상태였기 때문이다.
이 설정은 시스템이 새로운 패키지 목록을 확인하고,
업데이트가 승인없이 자동으로 설치하도록 미리 설정된 것이다.

```bash
sudo cat /etc/apt/apt.conf.d/20auto-upgrades
```

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

## 💡 해결

일단 이미 업데이트가 된 상태이기 때문에 기존 구버전 ChromeDriver를 제거하고
크롬 버전에 맞는 ChromeDriver를 수동으로 설치했다.

```bash
sudo rm $(which chromedriver)
wget https://edgedl.me.gvt1.com/edgedl/chrome/chrome-for-testing/137.0.7151.103/linux64/chromedriver-linux64.zip
unzip chromedriver-linux64.zip
sudo mv chromedriver-linux64/chromedriver /usr/bin/chromedriver
sudo chmod +x /usr/bin/chromedriver
```

설치가 완료되면 아래 명령어로 버전 확인을 시도한다:

```bash
chromedriver --version
```

그런데 예상치 못한 오류가 발생했다.

```
bash: /usr/local/bin/chromedriver: No such file or directory
```

과거에 설치했던 구버전 chromedriver가 `/usr/local/bin/chromedriver`에 있었고,
그걸 삭제한 후에도 Bash는 예전 경로를 캐시에 기억하고 있어서 여전히 그 경로를 실행하려 하기 때문이다.

이런 경우에는 아래와 같이 bash가 기억하던 실행 파일 경로 캐시를 초기화하면 된다.


```bash
hash -r
```

경로가 `/usr/bin/chromedriver`로 나오고, 버전이 크롬과 일치하면 설정이 잘 된 것이다.

```
which chromedriver
chromedriver --version
```

서버에서 자동으로 크롬이 업데이트되면 동일한 문제가 반복되기 때문에,
APT 패키지 시스템에서 특정 패키지의 업데이트를 차단하는 설정을 적용했다.

```bash
sudo apt-mark hold google-chrome-stable
```

이 명령어는 `google-chrome-stable` 패키지를 "hold" 상태로 만들어
향후 `apt upgrade`에서 자동으로 업데이트되지 않도록 막아준다.
즉, Chrome과 ChromeDriver 간 버전 불일치로 인한 문제를 예방할 수 있다.

필요할 경우 아래 명령으로 다시 hold 상태를 해제할 수 있다:

```bash
sudo apt-mark unhold google-chrome-stable
```

현재 hold 상태로 설정된 패키지는 아래 명령으로 확인 가능하다:

```bash
apt-mark showhold
```

## 마무리
이번 문제는 웹 기반 ETL 파이프라인을 구축하면서 Selenium으로 데이터를 수집하는 과정에서 발생했다.
크롤링을 자동화할 때, Chrome과 ChromeDriver의 버전 불일치로 인해 전체 파이프라인이 중단될 수 있는 리스크가 존재한다.

크롬은 APT를 통해 자동으로 업데이트되기 때문에,
크롬 브라우저의 버전을 의도적으로 고정(hold) 해 두는 것이 좋다.




