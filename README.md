# Python에서 Selenium Wire로 Web스크레이핑하기

[![Promo](https://github.com/bright-kr/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.co.kr/) 

이 가이드는 Web스크레이핑을 위해 Selenium Wire를 사용하는 방법을 설명하며, 요청 가로채기와 동적 프록시 ローテーション 같은 주제를 다룹니다.

- [Selenium Wire란?](#what-is-selenium-wire)
- [왜 Selenium Wire를 사용해야 하나요?](#why-use-selenium-wire)
- [Selenium Wire의 주요 기능](#key-features-of-selenium-wire)
    - [요청 및 응답 접근](#access-requests-and-responses)
    - [요청 및 응답 가로채기](#intercept-requests-and-responses)
    - [WebSocket 모니터링](#websocket-monitoring)
    - [프록시 관리](#manage-proxies)
- [Selenium Wire에서의 프록시 ローテーション](#proxy-rotation-in-selenium-wire)
    - [요구 사항](#requirements)
    - [1단계: 프록시 무작위화](#step-1-randomize-proxies)
    - [2단계: プロキ시 설정](#step-2-set-the-proxy)
    - [3단계: 대상 페이지 방문](#step-3-visit-the-target-page)
    - [4단계: 전체 코드로 결합](#step-4-put-it-all-together)
- [Bright Data Proxies로 ローテーティングプロキ시 사용하기](#a-better-approach-to-proxy-rotation-bright-data-proxies)
- [Web스크레이핑을 위한 Selenium vs Selenium Wire](#selenium-vs-selenium-wire-for-web-scraping)
- [결론](#conclusion)

## What Is Selenium Wire?

[Selenium Wire](https://github.com/wkeeling/selenium-wire)는 Selenium의 Python 바인딩을 위한 확장으로, 브라우저 요청에 대한 제어 기능을 제공합니다. Selenium을 사용하는 동안 Python 코드에서 직접 실시간으로 요청와 응답 모두를 가로채고 수정할 수 있습니다.

> **Note**:\
> 이 라이브러리는 더 이상 유지보수되지 않지만, 여러 스크레이핑 기술과 스크립트에서 여전히 사용하고 있습니다.

## Why Use Selenium Wire?

브라우저에는 Web스크레이핑을 어렵게 만들 수 있는 몇 가지 한계가 있습니다. 예를 들어, 인증이 포함된 プロキ시 URL을 설정하거나 [プロキ시를 ローテーション](/solutions/rotating-proxies)하는 것을 즉시(on the fly) 수행할 수 없습니다. Selenium Wire는 일반적인 인간 사용자처럼 사이트와 상호작용하면서 이러한 한계를 극복하도록 도와줍니다.

Web스크레이핑을 위해 Selenium Wire를 사용해야 하는 이유는 다음과 같습니다:

- **네트워크 트래픽에 직접 접근**: AJAX 요청 및 응답를 분석, 모니터링, 수정하여 유용한 데이터를 효율적으로 추출합니다.
- **안티봇 탐지 회피**: [`ChromeDriver`](https://developer.chrome.com/docs/chromedriver/downloads?hl=en)는 안티봇 시스템이 탐지에 사용하는 식별 가능한 세부 정보를 노출합니다. `undetected-chromedriver` 같은 기술은 Selenium Wire를 활용하여 이러한 정보를 마스킹하고 탐지 메커니즘을 우회합니다.
- **브라우저 유연성 향상**: 기존 브라우저는 고정된 시작 구성에 의존하므로 변경하려면 재시작이 필요합니다. Selenium Wire는 활성 세션 내에서 요청 헤더 및 プロキ시 설정을 실시간으로 업데이트할 수 있어, 동적 Web스크레이핑에 최적의 솔루션입니다.

## Key Features of Selenium Wire

### Access Requests and Responses

Selenium Wire는 브라우저의 HTTP/HTTPS 트래픽을 모니터링하고 캡처할 수 있게 하며, 다음과 같은 주요 속성에 접근할 수 있습니다:

| **Attribute** | **Description** |
| --- | --- |
| `driver.requests` | 캡처된 요청 목록을 시간 순서대로 보고합니다 |
| `driver.last_request` | 가장 최근에 캡처된 요청를 보고합니다  <br>(이는 `driver.requests[-1]`를 사용하는 것보다 더 효율적입니다) |
| `driver.wait_for_request(pat, timeout=10)` | 이 메서드는 `timeout` 매개변수로 정의된 시간 동안, `pat` 매개변수로 정의된 패턴(부분 문자열 또는 [정규식](/blog/web-data/web-scraping-with-regex) 가능)과 일치하는 요청를 볼 때까지 대기합니다. |
| `driver.har` | 발생한 HTTP 트랜잭션의 JSON 형식 [HAR](https://docs.brightdata.com/api-reference/proxy-manager/get_har_logs) 아카이브입니다. |
| `driver.iter_requests()` | 캡처된 요청에 대한 iterator를 반환합니다. |

Selenium Wire의 `Request` 객체는 다음 속성을 가집니다:

| **Attribute** | **Description** |
| --- | --- |
| `body` | 본문(body) 요청는 `bytes`로 제공됩니다. 요청에 body가 없으면 `body` 값은 비어 있게 됩니다(예: `b''`). |
| `cert` | 서버 SSL 인증서 정보를 dictionary 형식으로 보고합니다(HTTPS가 아닌 요청에서는 비어 있습니다). |
| `date` | 요청가 수행된 datetime을 표시합니다. |
| `headers` | 요청의 헤더에 대한 dictionary-like 객체를 보고합니다(Selenium Wire에서 헤더는 대소문자를 구분하지 않으며 중복이 허용됩니다). |
| `host` | 요청 host를 보고합니다(예: `https://brightdata.co.kr/`). |
| `method` | HHTP 메서드(`GET`, `POST` 등…)를 지정합니다 |
| `params` | 요청 매개변수의 dictionary를 보고합니다(같은 이름의 매개변수가 요청에 두 번 이상 나타나면 dictionary의 값은 list가 됩니다). |
| `path` | 요청 path를 보고합니다. |
| `querystring` | query string을 보고합니다. |
| `response` | リクエ스트와 연관된 응답 객체를 보고합니다(응답가 없는 リクエ스트의 경우 값은 `None`이 됩니다). |
| `url` | `host`, `path`, `querystring`을 포함한 완전한 リクエ스트 URL을 보고합니다. |
| `ws_messages` | 요청가 WebSocket인 경우(일반적으로 URL이 `wss://` 형태), `ws_messages`에 송수신된 websocket 메시지가 포함됩니다. |

반면 `Response` 객체는 다음 속성을 노출합니다:

| **Attribute** | **Description** |
| --- | --- |
| `body` | 본문(body) 응답는 `bytes`로 제공됩니다. 응답에 body가 없으면 `body` 값은 비어 있게 됩니다(예: `b''`). |
| `date` | 응답가 수신된 datetime을 표시합니다. |
| `headers` | 응답 헤더에 대한 dictionary-like 객체를 보고합니다(Selenium Wire에서 헤더는 대소문자를 구분하지 않으며 중복이 허용됩니다). |
| `reason` | `OK`, `Not Found` 등과 같은 응답 reason phrase를 보고합니다. |
| `status_code` | `200`, `404` 등과 같은 응답 상태를 보고합니다. |

이 기능을 테스트하기 위한 Python 스크립트를 작성해 보겠습니다:

```python
from seleniumwire import webdriver

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome()

try:
    # Open the target website
    driver.get("https://brightdata.co.kr/")

    # Access and print all captured requests
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Method: {request.method}")
        print(f"Headers: {request.headers}")
        print(f"Response Status Code: {request.response.status_code if request.response else 'No Response'}")
        print("-" * 50)

finally:
    # Close the browser
    driver.quit()
```

이 코드는 대상 웹사이트를 열고 `driver.requests`를 사용하여 요청를 캡처합니다. 그런 다음 for 루프를 통해 `url`, `method`, `headers` 같은 일부 요청 속성을 가로채어 출력합니다.

예상 결과는 다음과 같습니다:

![Some of the logged requests](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-98-1024x597.png)

### Intercept Requests and Responses


Selenium Wire는 네트워크 트래픽이 브라우저를 통과할 때 트리거되는 함수인 interceptor를 사용하여 요청와 응답를 가로채고 수정할 수 있게 해줍니다.

interceptor는 두 가지로 분리되어 있습니다:

* `driver.request_interceptor`: 요청를 가로채며 단일 인자를 받습니다.
* `driver.response_interceptor`: 응답를 가로채며, 원본 요청용 인자 1개와 응답용 인자 1개, 총 2개의 인자를 받습니다.

다음은 request interceptor를 사용하는 방법을 보여주는 예시입니다:

```python
from seleniumwire import webdriver

# Define the request interceptor function
def interceptor(request):
    # Add a custom header to all requests
    request.headers["X-Test-Header"] = "MyCustomHeaderValue"

    # Block requests to a specific domain
    if "example.com" in request.url:
        print(f"Blocking request to: {request.url}")
        request.abort()  # Abort the request

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome()

# Assign the interceptor function to the driver
driver.request_interceptor = interceptor

try:
    # Open a website that makes multiple requests
    driver.get("https://brightdata.co.kr/")

    # Print all captured requests
    for request in driver.requests:
        print(f"URL: {request.url}")
        print(f"Headers: {request.headers}")
        print("-" * 50)

finally:
    # Close the browser
    driver.quit()
```

이 스니펫이 수행하는 작업은 다음과 같습니다:

* **Interceptor 함수**: 모든 발신 요청마다 호출될 interceptor 함수를 생성합니다. `request.headers[]`로 모든 발신 요청에 커스텀 헤더를 추가합니다. 또한 `example.com` 도메인에 대한 브라우저 リクエ스트를 차단합니다.
* **요청 캡처**: 페이지가 로드된 뒤, 수정된 헤더를 포함하여 캡처된 모든 요청를 출력합니다.

> **Note:**\
>  리クエ스트 차단은 페이지가 광고, 분석 스크립트, 또는 타사 위젯처럼 작업에 필수적이지 않은 추가 리소스를 로드하는 경우 유용합니다. 이 접근 방식은 속도를 높이고 대역폭 소비를 최소화하여 스크레이핑 효율을 향상합니다.

예상 결과는 다음과 유사해야 합니다:

![Note the X-Test-Header](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-99-1024x538.png)

### WebSocket Monitoring

현대 웹사이트의 상당수는 서버와의 실시간 통신을 유지하기 위해 [`WebSockets`](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)에 의존합니다. 기존 HTTP 요청와 달리 `WebSockets`는 브라우저와 서버 사이에 연속 연결을 생성하여, 반복적인 핸드셰이크 없이 매끄러운 데이터 교환을 가능하게 합니다.  

중요한 데이터가 이러한 채널을 통해 흐르는 경우가 많으므로, `WebSocket` 트래픽을 가로채면 브라우저 기반 처리나 렌더링 없이도 실시간 서버 응답에 직접 접근할 수 있습니다.

다음은 Selenium Wire `WebSocket` 객체의 속성입니다:

| **Attribute** | **Description** |
| --- | --- |
| `content` | 메시지 content를 보고하며, `str` 또는 `bytes` 형식일 수 있습니다. |
| `date` | 메시지의 datetime을 표시합니다. |
| `headers` | 응답 헤더에 대한 dictionary-like 객체를 보고합니다(Selenium Wire에서 헤더는 대소문자를 구분하지 않으며 중복이 허용됩니다). |
| `from_client` | 메시지가 클라이언트에 의해 전송되었으면 `True`, 서버에 의해 전송되었으면 `False`를 반환하는 boolean입니다. |

### Manage Proxies

프록시 서버는 사용자 디바이스와 대상 웹사이트 사이의 중개자 역할을 하며 IP 주소를 숨깁니다. 이를 통해 IP 기반 제한을 우회하고, 속도 제한으로 인한 차단을 완화하며, 지리적으로 제한된 콘텐츠에 접근하여 원활한 Web스크레이핑을 가능하게 합니다.

Selenium Wire에서 프록시를 구성해 보겠습니다:

```python
# Set up Selenium Wire options
options = {
    "proxy": {
        "http": "<YOUR_HTTP_PROXY_URL>",
        "https": "<YOUR_HTTPS_PROXY_URL>"
    }
}

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome(seleniumwire_options=options)
```

이 설정은 기본 Selenium(vanilla Selenium)에서 プロキ시를 구성하는 방식과 다릅니다. 기본 Selenium에서는 Chrome의 `--proxy-server` 플래그에 의존해야 합니다. 이는 기본 Selenium에서 プロキ시 구성이 정적이라는 의미입니다. 한 번 プロ키시를 설정하면 전체 브라우저 세션 동안 계속 적용되며, 브라우저를 재시작하지 않고는 변경할 수 없습니다. 이러한 제한은 특히 동적 プロ키시 ローテーション이 필요한 경우 제약이 될 수 있습니다.

반면 Selenium Wire는 동일한 브라우저 인스턴스 내에서 プロ키시를 동적으로 변경할 수 있는 유연성을 제공합니다. 이는 `proxy` 속성 덕분에 가능합니다:

```python
# Dynamically change the proxy
driver.proxy = {
    "http": "<NEW_HTTP_PROXY_URL>",
    "https": "<NEW_HTTPS_PROXY_URL>"
}
```

또한 Chrome의 `--proxy-server` 플래그는 URL에 인증 자격 증명이 포함된 プロ키시를 지원하지 않습니다:

```
protocol://username:password@host:port
```

대신 Selenium Wire는 인증된 プロキ시를 완전히 지원하므로, Web스크레이핑에 더 나은 선택입니다.

## Proxy Rotation in Selenium Wire

이제 プロ키시 ローテーション을 위해 Selenium Wire 프로젝트를 설정해 보겠습니다. 이를 통해 각 요청마다 exit IP가 변경되도록 할 수 있습니다.

### Requirements

이 가이드의 이 부분을 따라 하려면 다음 사전 요구 사항이 필요합니다:

* Python 3.7 이상
* [지원되는 웹 브라우저](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/driver_location/)

먼저 가상 환경 디렉터리를 생성합니다:

```bash
python -m venv venv
```

Windows에서 활성화하려면 다음을 실행합니다:

```bash
venv\Scripts\activate
```

macOS/Linux에서는 다음을 실행합니다:

```bash
source venv/bin/activate
```

이제 Selenium Wire를 설치합니다(Selenium은 의존성으로 자동 설치됩니다):

```bash
pip install selenium-wire
```

### Step 1: Randomize Proxies

먼저 유효한 プロ키시 URL 목록이 필요합니다. [무료 プロ키시 목록](https://brightdata.co.kr/solutions/free-proxies)을 사용할 수 있습니다. 이를 list에 추가하고 [`random.choice()`](https://docs.python.org/3/library/random.html#random.choice)를 사용하여 무작위 요소를 선택합니다:

```python
def get_random_proxy():
    proxies = [
        "http://PROXY_1:PORT_NUMBER_X",
        "http://PROXY_2:PORT_NUMBER_Y",
        "http://PROXY_3:PORT_NUMBER_Z",
        # ...
    ]
    
    # Randomize the list
    return random.choice(proxies)
```

이 함수를 호출하면 목록에서 무작위 プロ키시 URL을 반환합니다.

작동하도록 하려면 `random`을 import하는 것을 잊지 마십시오:

```
import random
```

### Step 2: Set the Proxy

`get_random_proxy()` 함수를 호출하여 プロ키시 URL을 가져옵니다:

```python
proxy = get_random_proxy()
```

브라우저 인스턴스를 초기화하고 선택한 プロ키시를 설정합니다:

```python
# Selenium Wire configuration with the proxy
seleniumwire_options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Browser configuration
chrome_options = Options()
chrome_options.add_argument("--headless")  # Run the browser in headless mode 

# Initialize a browser instance with the given configurations
driver = webdriver.Chrome(service=Service(), options=chrome_options, seleniumwire_options=seleniumwire_options)
```

위 스니펫에는 다음 import가 필요합니다:

```python
from seleniumwire import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
```

브라우저 세션 동안 プロキ시를 동적으로 변경하려면 대신 다음 코드를 사용합니다:

```python
driver.proxy = {
    "http": proxy,
    "https": proxy
}
```

### Step 3: Visit the Target Page

대상 웹사이트를 방문하고, 출력을 추출한 다음, 브라우저를 종료합니다:

```python
try:
    # Visit the target page
    driver.get("https://httpbin.io/ip")

    # Extract the page output
    body = driver.find_element(By.TAG_NAME, "body").text
    print(body)
except Exception as e:
    # Handle any errors that occur with the browser or the proxy
    print(f"Error with proxy {proxy}: {e}")
finally:
    # Close the browser
    driver.quit()
```

작동하도록 하려면 Selenium에서 `By`를 import합니다:

```python
from selenium.webdriver.common.by import By
```

이 예시에서 목적지 페이지는 HTTPBin 프로젝트의 [`/ip`](https://httpbin.io/ip) 엔드포인트입니다. 이 페이지는 호출자의 IP 주소를 반환합니다. 모든 것이 예상대로라면, 스크립트는 실행할 때마다 プロ키시 목록의 서로 다른 IP를 출력해야 합니다.

### Step 4: Put It All Together

다음은 `selenium_wire.py` 파일에 들어가야 하는 전체 Selenium Wire プロ키시 ローテーション 로직입니다:

```python
import random
from seleniumwire import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By

def get_random_proxy():
    proxies = [
        "http://PROXY_1:PORT_NUMBER_X",
        "http://PROXY_2:PORT_NUMBER_Y",
        "http://PROXY_3:PORT_NUMBER_Z",
        # Add more proxies here...
    ]
    
    # Randomly pick a proxy
    return random.choice(proxies)
 
# Pick a random proxy URL 
proxy = get_random_proxy()

# Selenium Wire configuration with the proxy
seleniumwire_options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Browser configuration
chrome_options = Options()
chrome_options.add_argument("--headless")  # Run the browser in headless mode 

# Initialize a browser instance with the given configurations
driver = webdriver.Chrome(service=Service(), options=chrome_options, seleniumwire_options=seleniumwire_options)

try:
    # Visit the target page
    driver.get("https://httpbin.io/ip")

    # Extract the page output
    body = driver.find_element(By.TAG_NAME, "body").text
    print(body)
except Exception as e:
    # Handle any errors that occur with the browser or the proxy
    print(f"Error with proxy {proxy}: {e}")
finally:
    # Close the browser
    driver.quit()
```

파일을 실행하려면 다음을 실행합니다:

```bash
python3 selenium_wire.py
```

각 실행마다 출력은 다음과 같아야 합니다:

```json
{
  "origin": "PROXY_1:XXXX"
}
```

또는:

```json
{
  "origin": "PROXY_2:YYYY"
}
```

등등…

스크립트를 여러 번 실행하면 매번 다른 IP 주소가 표시되는 것을 확인할 수 있습니다.

## A Better Approach to Proxy Rotation: Bright Data Proxies

Selenium Wire에서 수동 プロ키시 ローテーション을 구현하려면 많은 boilerplate 코드가 필요하고, 유효한 プロ키시 URL 목록을 유지해야 합니다. 대신 IP 주소 변경을 자동으로 처리하는 Bright Data의 ローテーティング프록시를 사용할 수 있습니다. 사용 방법은 다음과 같습니다.

이미 계정이 있다면 Bright Data에 로그인합니다. 그렇지 않다면 무료로 계정을 생성하십시오. 그러면 다음 사용자 대시보드에 접근할 수 있습니다:

![The Bright Data dashboard](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-100-1024x498.png)

“View proxy products” 버튼을 클릭합니다:

![View proxy products](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-101.png)

아래의 “Proxies & Scraping Infrastructure” 페이지로 리디렉션됩니다:

![Configuring your residential proxies](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-102-1024x483.png)

아래로 스크롤하여 “[Residential Proxies](/blog/proxy-101/ultimate-guide-to-proxy-types)” 카드로 이동한 다음 “Get started” 버튼을 클릭합니다:

![Residential proxies](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-103.png)

레ジデンシャルプロキ시 구성 대시보드로 이동합니다. 안내되는 wizard를 따라 필요에 맞게 プロ키시 서비스를 설정하십시오.

![Configuring your residential proxies](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-104.png)

“Access parameters” 탭으로 이동하여 다음과 같이 プロ키시의 host, port, username, password를 가져옵니다:

![access parameter](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-105.png)

“Host” 필드에는 이미 port가 포함되어 있다는 점에 유의하십시오.

이것이 プロ키시 URL을 구성하고 Selenium Wire에 설정하는 데 필요한 전부입니다. 모든 정보를 모아 다음 구문으로 URL을 구성합니다:

```
<username>:<password>@<host>
```

예를 들어 이 경우 다음과 같습니다:

```
brd-customer-hl_4hgu8dwd-zone-residential:[email protected]:XXXXX
```

“Active proxy”를 토글하고 마지막 안내를 따르면 완료됩니다!

![Active proxy toggle](https://github.com/bright-kr/selenium-wire-web-scraping/blob/main/Images/image-106-1024x164.png)

다음은 Bright Data 통합을 위한 Selenium Wire プロ키시 스니펫입니다:

```python
# Bright Data proxy URL
proxy = "brd-customer-hl_4hgu8dwd-zone-residential:[email protected]:XXXXX"

# Set up Selenium Wire options
options = {
    "proxy": {
        "http": proxy,
        "https": proxy
    }
}

# Initialize the WebDriver with Selenium Wire
driver = webdriver.Chrome(seleniumwire_options=options)
```

## Selenium vs Selenium Wire for Web Scraping

요약하면, 다음은 Selenium과 Selenium Wire의 비교입니다:

|     | **Selenium** | **Selenium Wire** |
| --- | --- | --- |
| **Purpose** | UI 테스트 및 웹 상호작용을 수행하기 위해 웹 브라우저를 자동화합니다 | HTTP/HTTPS 요청 및 응답를 검사하고 수정하기 위한 추가 기능을 제공하도록 Selenium을 확장합니다 |
| **HTTP/HTTPS request handling** | HTTP/HTTPS リクエ스트 또는 응답에 대한 직접 접근을 제공하지 않습니다 | HTTP/HTTPS リクエ스트 및 응답의 검사, 수정, 캡처를 허용합니다 |
| **Proxy support** | 제한적인 プロ키시 지원(수동 구성이 필요함) | 동적 설정을 지원하는 고급 プロ키시 관리 |
| **Performance** | 가볍고 빠릅니다 | 네트워크 트래픽의 캡처 및 처리 때문에 약간 더 느립니다 |
| **Use cases** | 주로 웹 애플리케이션의 기능 테스트에 사용되며, 기본적인 Web스크레이핑 케이스에 유용합니다 | API 테스트, 네트워크 트래픽 디버깅, Web스크레이핑에 유용합니다 |

## Conclusion

Selenium Wire는 Web스크레이핑에 효율적으로 사용할 수 있지만, 유지보수되지 않는 소프트웨어이며 모든 경우에 통용되는(one-size-fits-all) 솔루션은 아닙니다.

대신 [Bright Data의 Scraping Browser](https://brightdata.co.kr/products/scraping-browser) 같은 전용 스クレ이ピング 브라우저와 함께 기본 Selenium을 사용하는 것을 고려하십시오. 이는 [Playwright](https://brightdata.co.kr/products/scraping-browser/playwright), [Puppeteer](https://brightdata.co.kr/products/scraping-browser/puppeteer), [Selenium](https://brightdata.co.kr/products/scraping-browser/selenium) 등과 함께 동작하는 확장 가능한 클라우드 브라우저입니다. 또한 브라우ザフィンガープリント 관리, 재시도, CAPTCHA 해결 등을 수행하는 동시에 각 요청마다 exit IP를 원활하게 ローテーション합니다. 이를 사용해 차단 문제를 제거하고 스크레이핑 워크플로를 최적화해 보십시오.