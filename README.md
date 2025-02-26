# 从 Google Travel 抓取酒店信息

[![Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.com/) 

本指南讲解了如何使用 Selenium 方法或 Bright Data 的 API 来从 Google Travel 获取酒店列表、价格和酒店设施信息。

- [环境准备](#环境准备)
- [从 Google Travel 获取哪些信息](#从-google-travel-获取哪些信息)
- [使用 Selenium 来提取数据](#使用-selenium-来提取数据)
- [使用 Bright Data 的 Travel API 来提取数据](#使用-bright-data-的-travel-api-来提取数据)
    - [使用 Requests](#使用-requests)
    - [使用 AIOHTTP](#使用-aiohttp)
- [Bright Data 的其他解决方案](#bright-data-的其他解决方案)

## 环境准备

要抓取旅游数据，你需要 Python 和任意以下模块之一：Selenium、Requests 或 AIOHTTP。  
如果使用 Selenium，你可以直接从 Google Travel 抓取酒店信息。  
如果使用 Requests 或 AIOHTTP，则需通过 Bright Data 的 [Booking.com API](https://www.bright.cn/products/web-scraper/booking) 获取数据。

若选择使用 Selenium，请确保安装了 [webdriver](https://googlechromelabs.github.io/chrome-for-testing/)。如果你不熟悉 Selenium，可以先阅读 [这篇指南](https://www.bright.cn/blog/how-tos/using-selenium-for-web-scraping)，快速上手。

安装 Selenium:

```
pip install selenium
```

安装 Requests:

```
pip install requests
```

安装 AIOHTTP:

```bash
pip install aiohttp
```

## 从 Google Travel 获取哪些信息

所有酒店结果都嵌在 Google Travel 自定义的 `c-wiz` 元素中。

![Inspect c-wiz Element](https://brightdata.com/wp-content/uploads/2025/01/image-32.png)

但在页面上有许多 `c-wiz` 元素。每个酒店卡片都包含一个直接从某个 `div` 和 `c-wiz` 元素继承的 `a` 元素。我们可以写一个 CSS 选择器来获取所有从这些元素继承的 `a` 标签：`c-wiz > div > a`。

![Inspect a Element](https://brightdata.com/wp-content/uploads/2025/01/image-33.png)

酒店名称嵌在一个 `h2` 元素中。

![Inspect h2 Element](https://brightdata.com/wp-content/uploads/2025/01/image-34.png)

价格信息则嵌在一个 `span` 元素中。

![Inspect Price Element](https://brightdata.com/wp-content/uploads/2025/01/image-35.png)

酒店设施（amenities）则是嵌在 `li` 列表元素中。

![Inspect Amenities](https://brightdata.com/wp-content/uploads/2025/01/image-36.png)

获取到一个酒店卡片后，我们可以依次提取上述信息。

## 使用 Selenium 来提取数据

一旦知道目标元素，使用 Selenium 进行数据提取就相对直接了。然而，由于 Google Travel 会动态加载结果，需要格外小心，需依赖适当的等待时间、鼠标点击以及自定义窗口等才能让流程顺利进行。

下面是完整的 Python 脚本示例：

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.action_chains import ActionChains
import json
from time import sleep

OPTIONS = webdriver.ChromeOptions()
OPTIONS.add_argument("--headless")
OPTIONS.add_argument("--window-size=1920,1080")



def scrape_hotels(location, pages=5):
    driver = webdriver.Chrome(options=OPTIONS)
    actions = ActionChains(driver)
    url = f"https://www.google.com/travel/search?q={location}"
    driver.get(url)
    done = False

    found_hotels = []
    page = 1
    result_number = 1
    while page <= pages:
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        sleep(5)
        hotel_links = driver.find_elements(By.CSS_SELECTOR, "c-wiz > div > a")
        print(f"-----------------PAGE {page}------------------")
        print("FOUND ITEMS: ", len(hotel_links))
        for hotel_link in hotel_links:
            hotel_card = hotel_link.find_element(By.XPATH, "..")
            try:
                info = {}
                info["url"] = hotel_link.get_attribute("href")
                info["rating"] = 0.0
                info["price"] = "n/a"
                info["name"] = hotel_card.find_element(By.CSS_SELECTOR, "h2").text
                price_holder = hotel_card.find_elements(By.CSS_SELECTOR, "span")
                info["amenities"] = []
                amenities_holders = hotel_card.find_elements(By.CSS_SELECTOR, "li")
                for amenity in amenities_holders:
                    info["amenities"].append(amenity.text)
                if "DEAL" in price_holder[0].text or "PRICE" in price_holder[0].text:
                    if price_holder[1].text[0] == "$":
                        info["price"] = price_holder[1].text
                else:
                    info["price"] = price_holder[0].text
                rating_holder = hotel_card.find_elements(By.CSS_SELECTOR, "span[role='img']")
                if rating_holder:
                    info["rating"] = float(rating_holder[0].get_attribute("aria-label").split(" ")[0])
                info["result_number"] = result_number
                
                if info not in found_hotels:
                    found_hotels.append(info)
                result_number+=1
                
            except:
                continue
        print("Scraped Total:", len(found_hotels))
        
        next_button = driver.find_elements(By.XPATH, "//span[text()='Next']")
        if next_button:
            print("next button found!")
            sleep(1)
            actions.move_to_element(next_button[0]).click().perform()
            page+=1
            sleep(5)
        else:
            done = True

    driver.quit()

    with open("scraped-hotels.json", "w") as file:
        json.dump(found_hotels, file, indent=4)

if __name__ == "__main__":
    PAGES = 2
    scrape_hotels("miami", pages=PAGES)
```

接下来分步骤说明脚本的作用：

1. 首先，我们创建了一个 `ChromeOptions` 实例，并使用它添加了 `--headless` 和 `--window-size=1920,1080` 参数。

> **注意**  
> 如果不设置自定义窗口大小，页面可能无法正确加载，导致重复抓取相同结果。

2. 当我们启动浏览器时，使用 `options=OPTIONS` 传递自定义选项。这会用自定义参数启动 Chrome。

3. `ActionChains(driver)` 为我们创建一个操作链实例，稍后我们会用它将鼠标移动到 “Next” 按钮并点击。

4. 我们用一个 `while` 循环控制整个抓取的流程，一旦抓取完成就退出循环。

5. `hotel_links = driver.find_elements(By.CSS_SELECTOR, "c-wiz > div > a")` 获取页面上所有酒店的链接。我们使用 xpath 获取它们的父元素：`hotel_card = hotel_link.find_element(By.XPATH, "..")`。

6. 依次提取想要的信息：
    - url: `hotel_link.get_attribute("href")`
    - name: `hotel_card.find_element(By.CSS_SELECTOR, "h2").text`
    - 当查找价格时，有时卡片里会出现其他元素，例如 `DEAL` 或 `GREAT PRICE`。为了确保拿到真正的价格，我们会获取所有的 `span` 元素组成的数组。如果该数组包含这些单词，那就取第二个元素（`price_holder[1].text`），否则就取第一个元素（`price_holder[0].text`）。
    - 查找评分时，我们使用 `find_elements()` 方法。若没有评分，就给一个默认值 `n/a`。
    - `hotel_card.find_elements(By.CSS_SELECTOR, "li")` 可以获取所有设施列表。我们使用它们的 `text` 属性来提取。
7. 这个循环会一直进行，直到我们抓取了目标页数的所有数据或者没有可前进的页面，结束后将 `done` 设置为 `True`，跳出循环。
8. 最后关闭浏览器，使用 `json.dump()` 将数据保存到 JSON 文件中。

## 使用 Bright Data 的 Travel API 来提取数据

如果你不想依赖爬虫或处理各种选择器与定位器，可以利用 [Travel Data](https://www.bright.cn/use-cases/travel) 或者配合 [Booking.com API](https://www.bright.cn/products/web-scraper/booking) 来获取酒店数据。这里演示两种方式：`requests` 以及 AIOHTTP。

### 使用 Requests

下面的代码用于演示如何在 Booking.com API 中实现数据获取。只需填入 API key、目标位置、入住和离店的日期。流程是先向 API 发出请求生成数据，然后每隔 10 秒轮询一次，直到返回的数据准备就绪，最后将结果保存为 JSON 文件。

```python
import requests
import json
import time


def get_bookings(api_key, location, dates):
    url = "https://api.brightdata.com/datasets/v3/trigger"

    #booking.com dataset
    dataset_id = "gd_m4bf7a917zfezv9d5"

    endpoint = f"{url}?dataset_id={dataset_id}&include_errors=true"
    auth_token = api_key

    #
    headers = {
        "Authorization": f"Bearer {auth_token}",
        "Content-Type": "application/json"
    }

    payload = [
        {
            "url": "https://www.booking.com",
            "location": location,
            "check_in": dates["check_in"],
            "check_out": dates["check_out"],
            "adults": 2,
            "rooms": 1
        }
    ]

    response = requests.post(endpoint, headers=headers, json=payload)

    if response.status_code == 200:
        print("Request successful. Response:")
        print(json.dumps(response.json(), indent=4))
        return response.json()["snapshot_id"]
    else:
        print(f"Error: {response.status_code}")
        print(response.text)

def poll_and_retrieve_snapshot(api_key, snapshot_id, output_file="snapshot-data.json"):
    #create the snapshot url
    snapshot_url = f"https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}?format=json"
    headers = {
        "Authorization": f"Bearer {api_key}"
    }

    print(f"Polling snapshot for ID: {snapshot_id}...")

    while True:
        response = requests.get(snapshot_url, headers=headers)
        
        if response.status_code == 200:
            print("Snapshot is ready. Downloading...")
            snapshot_data = response.json()
            #write the snapshot to a new json file
            with open(output_file, "w", encoding="utf-8") as file:
                json.dump(snapshot_data, file, indent=4)
            print(f"Snapshot saved to {output_file}")
            break
        elif response.status_code == 202:
            print("Snapshot is not ready yet. Retrying in 10 seconds...")
        else:
            print(f"Error: {response.status_code}")
            print(response.text)
            break
        
        time.sleep(10)


if __name__ == "__main__":
    
    API_KEY = "your-bright-data-api-key"
    LOCATION = "Miami"
    CHECK_IN = "2025-02-01T00:00:00.000Z"
    CHECK_OUT = "2025-02-02T00:00:00.000Z"
    DATES = {
        "check_in": CHECK_IN,
        "check_out": CHECK_OUT
    }
    snapshot_id = get_bookings(API_KEY, LOCATION, DATES)
    poll_and_retrieve_snapshot(API_KEY, snapshot_id)
```

- `get_bookings()` 接受 `API_KEY`、`LOCATION` 和 `DATES`，请求返回一个 `snapshot_id`。
- 这个 `snapshot_id` 用来获取具体的快照数据。
- `poll_and_retrieve_snapshot()` 会每隔 10 秒轮询一次，直到数据准备就绪。
- 数据就绪后，用 `json.dump()` 将其保存为一个 JSON 文件。

运行以上代码后，终端输出大致如下：

```
Request successful. Response:
{
    "snapshot_id": "s_m5moyblm1wikx4ntot"
}
Polling snapshot for ID: s_m5moyblm1wikx4ntot...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is not ready yet. Retrying in 10 seconds...
Snapshot is ready. Downloading...
Snapshot saved to snapshot-data.json
```

随后你会得到一个包含许多对象的 JSON 文件。例如：

```json
{
        "input": {
            "url": "https://www.booking.com",
            "location": "Miami",
            "check_in": "2025-02-01T00:00:00.000Z",
            "check_out": "2025-02-02T00:00:00.000Z",
            "adults": 2,
            "rooms": 1
        },
        "url": "https://www.booking.com/hotel/us/ramada-plaze-by-wyndham-marco-polo-beach-resort.html?checkin=2025-02-01&checkout=2025-02-02&group_adults=2&no_rooms=1&group_children=",
        "location": "Miami",
        "check_in": "2025-02-01T00:00:00.000Z",
        "check_out": "2025-02-02T00:00:00.000Z",
        "adults": 2,
        "children": null,
        "rooms": 1,
        "id": "55989",
        "title": "Ramada Plaza by Wyndham Marco Polo Beach Resort",
        "address": "19201 Collins Avenue",
        "city": "Sunny Isles Beach (Florida)",
        "review_score": 6.2,
        "review_count": "1788",
        "image": "https://cf.bstatic.com/xdata/images/hotel/square600/414501733.webp?k=4c14cb1ec5373f40ee83d901f2dc9611bb0df76490f3673f94dfaae8a39988d8&o=",
        "final_price": 217,
        "original_price": 217,
        "currency": "USD",
        "tax_description": null,
        "nb_livingrooms": 0,
        "nb_kitchens": 0,
        "nb_bedrooms": 0,
        "nb_all_beds": 2,
        "full_location": {
            "description": "This is the straight-line distance on the map. Actual travel distance may vary.",
            "main_distance": "11.4 miles from downtown",
            "display_location": "Miami Beach",
            "beach_distance": "Beachfront",
            "nearby_beach_names": []
        },
        "no_prepayment": false,
        "free_cancellation": true,
        "property_sustainability": {
            "is_sustainable": false,
            "level_id": "L0",
            "facilities": [
                "436",
                "490",
                "492",
                "496",
                "506"
            ]
        },
        "timestamp": "2025-01-07T16:43:24.954Z"
    },
```

### 使用 AIOHTTP

使用 [AIOHTTP](https://www.bright.cn/blog/web-data/speed-up-web-scraping) 库，可以同时触发、轮询并下载多个数据集，从而提速。下面的示例基于 Requests 示例的思路，用 `aiohttp.ClientSession()` 实现异步请求。

```python
import aiohttp
import asyncio
import json


async def get_bookings(api_key, location, dates):
    url = "https://api.brightdata.com/datasets/v3/trigger"
    dataset_id = "gd_m4bf7a917zfezv9d5"
    endpoint = f"{url}?dataset_id={dataset_id}&include_errors=true"
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }
    payload = [
        {
            "url": "https://www.booking.com",
            "location": location,
            "check_in": dates["check_in"],
            "check_out": dates["check_out"],
            "adults": 2,
            "rooms": 1
        }
    ]

    async with aiohttp.ClientSession(headers=headers) as session:
        async with session.post(endpoint, json=payload) as response:
            if response.status == 200:
                response_data = await response.json()
                print(f"Request successful for location: {location}. Response:")
                print(json.dumps(response_data, indent=4))
                return response_data["snapshot_id"]
            else:
                print(f"Error for location: {location}. Status: {response.status}")
                print(await response.text())
                return None


async def poll_and_retrieve_snapshot(api_key, snapshot_id, output_file):
    snapshot_url = f"https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}?format=json"
    headers = {
        "Authorization": f"Bearer {api_key}"
    }

    print(f"Polling snapshot for ID: {snapshot_id}...")

    async with aiohttp.ClientSession(headers=headers) as session:
        while True:
            async with session.get(snapshot_url) as response:
                if response.status == 200:
                    print(f"Snapshot for {output_file} is ready. Downloading...")
                    snapshot_data = await response.json()
                    # Save snapshot data to a file
                    with open(output_file, "w", encoding="utf-8") as file:
                        json.dump(snapshot_data, file, indent=4)
                    print(f"Snapshot saved to {output_file}")
                    break
                elif response.status == 202:
                    print(f"Snapshot for {output_file} is not ready yet. Retrying in 10 seconds...")
                else:
                    print(f"Error polling snapshot for {output_file}. Status: {response.status}")
                    print(await response.text())
                    break

            await asyncio.sleep(10)


async def process_location(api_key, location, dates):
    snapshot_id = await get_bookings(api_key, location, dates)
    if snapshot_id:
        output_file = f"snapshot-{location.replace(' ', '_').lower()}.json"
        await poll_and_retrieve_snapshot(api_key, snapshot_id, output_file)


async def main():
    api_key = "your-bright-data-api-key"
    locations = ["Miami", "Key West"]
    dates = {
        "check_in": "2025-02-01T00:00:00.000Z",
        "check_out": "2025-02-02T00:00:00.000Z"
    }

    # Process all locations in parallel
    tasks = [process_location(api_key, location, dates) for location in locations]
    await asyncio.gather(*tasks)


if __name__ == "__main__":
    asyncio.run(main())
```

- `get_bookings()` 和 `poll_and_retrieve_snapshot()` 都使用 `aiohttp.ClientSession` 对象异步向服务器请求。
- `process_location()` 用于处理单个地点的数据抓取流程。
- `main()` 可以一次性并行处理多个地点。

输出示例如下：

```
Request successful for location: Miami. Response:
{
    "snapshot_id": "s_m5mtmtv62hwhlpyazw"
}
Request successful for location: Key West. Response:
{
    "snapshot_id": "s_m5mtmtv72gkkgxvdid"
}
Polling snapshot for ID: s_m5mtmtv62hwhlpyazw...
Polling snapshot for ID: s_m5mtmtv72gkkgxvdid...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-miami.json is ready. Downloading...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot saved to snapshot-miami.json
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is not ready yet. Retrying in 10 seconds...
Snapshot for snapshot-key_west.json is ready. Downloading...
Snapshot saved to snapshot-key_west.json
```

## Bright Data 的其他解决方案

除了 [Web Scraper API](https://www.bright.cn/products/web-scraper)，Bright Data 还提供了可直接使用的数据集，满足多种需求。最受欢迎的旅游类数据集包括：

- [酒店数据集](https://www.bright.cn/products/datasets/travel/hotels)  
- [Expedia 数据集](https://www.bright.cn/products/datasets/travel/expedia)  
- [旅游数据集](https://www.bright.cn/products/datasets/tourism)  
- [Booking.com 数据集](https://www.bright.cn/products/datasets/booking)  
- [TripAdvisor 数据集](https://www.bright.cn/products/datasets/tripadvisor)

你可以选择全托管或自托管的定制数据集方案，从任意公共网站抓取数据，并按照你的需求进行定制化处理。
