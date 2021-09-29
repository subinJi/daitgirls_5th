# 파이썬을 이용한 아마존 리뷰 크롤링


## 1. 실행 부분 코드
```python3
import math
import re
import csv
from bs4 import BeautifulSoup
from selenium import webdriver

ASIN = 'B072BCNRTY'
all_reviews = amazon_review_crawling(ASIN)  # dictionary를 가지고 있는 list를 반환함
create_csv(ASIN)  # csv파일로 크롤링한 데이터 저장
```
<br>

## 2. 한 페이지를 크롤링 해오는 함수
```python3
def onepage_crawling(div):
    lst = []  # 한 페이지의 리뷰를 담는 list
    titles = div.findAll('a', {'data-hook', 'review-title'})  # 타이틀
    ratings = div.findAll('span', {'class', 'a-icon-alt'})  # 별점
    dates = div.findAll('span', {'data-hook', 'review-date'})  # 날짜
    contents = div.findAll('span', {'class', 'review-text'})  # 내용

    # 현재 열려있는 페이지의 리뷰를 리스트에 하나씩 추가해줌
    for title, rating, date, content in zip(titles, ratings, dates, contents):
        title = title.span.text  # 타이틀
        rating = rating.text.split(' ')[-1]  # 별점
        
        # 날짜
        date = date.text.split(' ')[1:-1]  # ['2021년', '8월', '24일']
        year = re.sub('[^0-9]', '', date[0])  # 년
        month = re.sub('[^0-9]', '', date[1])  # 월
        month = '0' + month if len(month) == 1 else month  # 8월처럼 한자리수인 경우 08로 만들어줌
        day = re.sub('[^0-9]', '', date[2])  # 일
        review_date = f'{year}/{month}/{day}'  # ex) '2021/8/24'
        
        content = content.text.replace('\n\n', '').lstrip()  # 내용

        # 리스트에 리뷰 하나를 추가해줌
        lst.append({
            "title": title,
            "rating": rating,
            "date": review_date,
            "content": content
        })
        
        return lst
```
<br>

## 3. 본격 크롤링 시작
```python3
def amazon_review_crawling(ASIN):
    lst = []
    
    # 우리가 사용할 url
    url = f'https://www.amazon.com/Julius-Studio-Background-Photography-JSAG283/product-reviews/{ASIN}/ref=cm_cr_arp_d_viewopt_srt?ie=UTF8&reviewerType=all_reviews&sortBy=recent&pageNumber='

    # ------------------------------ 크롤링 설정 -----------------------------------
    options = webdriver.ChromeOptions()  # options에 webdriver의 크롬옵션을 할당해줌
    options.add_argument('headless')  # 크롬 창을 띄우지 않고 동작하게 해줌
    options.add_argument('window-size=1920x1080')  # 해상도 1920x1080의 크기로 크롤링 하게됨
    options.add_argument("disable-gpu")  # gpu 사용안하게 설정해줌

    # executable_path : 크롬드라이버가 깔려있는 경로
    # options : 위에서 설정한 옵션을 적용해줌
    driver = webdriver.Chrome(executable_path='c:/chromedriver.exe', options=options)

    driver.implicitly_wait(3)  # ddos공격으로 의심할 수 있으므로 3초 간격으로 크롤링해줌
    driver.get(url + '1')  # 크롤링할 페이지 url 끝에 '1'을 붙여줌

    soup = BeautifulSoup(driver.page_source, 'html.parser')
    # ----------------------------------------------------------------------------

    # 크롤링 해야하는 총 페이지 수
    MAX_PAGENUM = math.ceil(int(re.sub(r'[^0-9]', '', soup.find('div', id='filter-info-section').span.text.split('|')[1])) / 10)

    # 페이지별로 크롤링 시작!
    for page_num in range(1, 11):  # MAX_PAGENUM+1을 해주어야 하지만 오래걸리는 관계로 10페이지만 크롤링 하게 해놓음
        driver.get(url + str(page_num))  # 크롤링할 페이지 끝에 page_num을 붙여줌
        soup = BeautifulSoup(driver.page_source, 'html.parser')
        reviews_div = soup.find('div', id='cm_cr-review_list')  # 우리가 가져올 리뷰 리스트 div
        reviews = onepage_crawling(reviews_div)  # 한 페이지씩 크롤링해줌(dictionary 반환)
        lst.extend(reviews)
        
        print(f'총 {MAX_PAGENUM}중 {page_num}페이지 완료됨')
        

    driver.quit()  # 크롤링 끝났으니 드라이버 종료!

    return lst
```
<br>

## 4. 크롤링한 데이터를 csv파일로 만들어 저장함
```python3
def create_csv(fname):
    with open(f'./{fname}.csv', 'w', encoding='UTF-8', newline='') as f:
        makewrite = csv.writer(f)

        makewrite.writerow(['Title', 'Rating', 'Date', 'Content'])
        for value in all_reviews:
            makewrite.writerow(value.values())
```
<br><br>

## 전체 코드
```python3
import math
import re
import csv
from bs4 import BeautifulSoup
from selenium import webdriver


# 한 페이지를 크롤링 해오는 함수
def onepage_crawling(div):
    lst = []  # 한 페이지의 리뷰를 담는 list
    titles = div.findAll('a', {'data-hook', 'review-title'})  # 타이틀
    ratings = div.findAll('span', {'class', 'a-icon-alt'})  # 별점
    dates = div.findAll('span', {'data-hook', 'review-date'})  # 날짜
    contents = div.findAll('span', {'class', 'review-text'})  # 내용

    # 현재 열려있는 페이지의 리뷰를 리스트에 하나씩 추가해줌
    for title, rating, date, content in zip(titles, ratings, dates, contents):
        title = title.span.text  # 타이틀
        rating = rating.text.split(' ')[-1]  # 별점
        
        # 날짜
        date = date.text.split(' ')[1:-1]  # ['2021년', '8월', '24일']
        year = re.sub('[^0-9]', '', date[0])  # 년
        month = re.sub('[^0-9]', '', date[1])  # 월
        month = '0' + month if len(month) == 1 else month  # 8월처럼 한자리수인 경우 08로 만들어줌
        day = re.sub('[^0-9]', '', date[2])  # 일
        review_date = f'{year}/{month}/{day}'  # ex) '2021/8/24'
        
        content = content.text.replace('\n\n', '').lstrip()  # 내용

        # 리스트에 리뷰 하나를 추가해줌
        lst.append({
            "title": title,
            "rating": rating,
            "date": review_date,
            "content": content
        })
        
        return lst


# ---------------------------------------------------------------------------------

# 본격 크롤링 시작
def amazon_review_crawling(ASIN):
    lst = []
    
    # 우리가 사용할 url
    url = f'https://www.amazon.com/Julius-Studio-Background-Photography-JSAG283/product-reviews/{ASIN}/ref=cm_cr_arp_d_viewopt_srt?ie=UTF8&reviewerType=all_reviews&sortBy=recent&pageNumber='

    # ------------------------------ 크롤링 설정 -----------------------------------
    options = webdriver.ChromeOptions()  # options에 webdriver의 크롬옵션을 할당해줌
    options.add_argument('headless')  # 크롬 창을 띄우지 않고 동작하게 해줌
    options.add_argument('window-size=1920x1080')  # 해상도 1920x1080의 크기로 크롤링 하게됨
    options.add_argument("disable-gpu")  # gpu 사용안하게 설정해줌

    # executable_path : 크롬드라이버가 깔려있는 경로
    # options : 위에서 설정한 옵션을 적용해줌
    driver = webdriver.Chrome(executable_path='c:/chromedriver.exe', options=options)

    driver.implicitly_wait(3)  # ddos공격으로 의심할 수 있으므로 3초 간격으로 크롤링해줌
    driver.get(url + '1')  # 크롤링할 페이지 url 끝에 '1'을 붙여줌

    soup = BeautifulSoup(driver.page_source, 'html.parser')
    # ----------------------------------------------------------------------------

    # 크롤링 해야하는 총 페이지 수
    MAX_PAGENUM = math.ceil(int(re.sub(r'[^0-9]', '', soup.find('div', id='filter-info-section').span.text.split('|')[1])) / 10)

    # 페이지별로 크롤링 시작!
    for page_num in range(1, 11):  # MAX_PAGENUM+1을 해주어야 하지만 오래걸리는 관계로 10페이지만 크롤링 하게 해놓음
        driver.get(url + str(page_num))  # 크롤링할 페이지 끝에 page_num을 붙여줌
        soup = BeautifulSoup(driver.page_source, 'html.parser')
        reviews_div = soup.find('div', id='cm_cr-review_list')  # 우리가 가져올 리뷰 리스트 div
        reviews = onepage_crawling(reviews_div)  # 한 페이지씩 크롤링해줌(dictionary 반환)
        lst.extend(reviews)
        
        print(f'총 {MAX_PAGENUM}중 {page_num}페이지 완료됨')
        

    driver.quit()  # 크롤링 끝났으니 드라이버 종료!

    return lst


def create_csv(fname, lst):
    with open(f'./{fname}.csv', 'w', encoding='UTF-8', newline='') as f:
        makewrite = csv.writer(f)

        makewrite.writerow(['Title', 'Rating', 'Date', 'Content'])
        for value in lst:
            makewrite.writerow(value.values())


ASIN = 'B072BCNRTY'
all_reviews = amazon_review_crawling(ASIN)  # dictionary를 가지고 있는 list를 반환함
create_csv(ASIN, all_reviews)  # csv파일로 크롤링한 데이터 저장
```
