## 프로젝트 개요
- Bosch 세탁기의 고객 리뷰를 통해 사용자 요구사항(Customer Requirements, CR)을 파악하기 위해 리뷰 수집 필요
- Amazon 리뷰 데이터를 기반으로 실제 사용자의 평가 내용을 수집
- 수집한 리뷰는 자연어 처리 기반으로 전처리하여, 명사 추출, 표제어 변환 등을 통해 분석에 적합한 형태로 가공
- 전처리된 데이터는 LDA 기반 토픽 모델링, 키워드 기반 검색, 시각화 등에 활용
  - LDA 기반 토픽 모델링 : 리뷰에서 핵심 단어들을 기반으로 주제를 추출
  - 키워드 기반 검색 : 키워드와 관련된 사용자들의 리뷰를 필터링하여 비교분석하기 위해 활용
       - 시각화 : 파이차트를 통해 별점 비율 확인 가능

해당 레포지토리는 **리뷰 수집**을 위한 **크롤링 코드/시각화 코드**를 정리한 저장소입니다.

## 데이터

### 수집 대상
- **웹사이트**: [Amazon.com](https://www.amazon.com)
- **대상**: Bosch 세탁기 WAJ2416WIN 모델의 리뷰 (https://www.amazon.in/product-reviews/B08SR52BSM/ref=cm_cr_arp_d_paging_btm_next_2?ie=UTF8&reviewerType=all_reviews&pageNumber=1)
- **수집 기간**: 2015년 5월 ~ 2023년 3월
![image](https://github.com/user-attachments/assets/5ceaa9f0-f848-4527-9dd6-33b4996da562)

### 수집 방식
- Python 기반 웹 크롤링(Selenium + BeautifulSoup) 사용
- 수집 항목: 리뷰 제목, 본문, 별점, 날짜

### 예시

| 날짜        | 별점 | 제목                  | 본문         |
|-------------|------|------------------------|--------------|
| 2023-02-20  | 5    | Durable Solid excellent | Superlike    |
| 2023-02-14  | 5    | Great quality          | Fantastic    |
| ...         | ...  | ...                    | ...          |
|2019-11-05|5|Less noise	|Less noise	|
|...|...|...|...|

> 크롤링 코드는 `code/1_crawling_Bosch.ipynb`

---

리뷰 수집, 전처리, 분석 코드
===============

**주요 파일 목록**
**실행 순서**
---------
1. 리뷰 수집: 1_crawling_Bosch_(WAJ2416WIN).ipynb
2. 데이터 전처리: 2_review_preprocessing.ipynb
3. 토픽 모델링: 3_review_LDA.ipynb
4. 검색 및 시각화: 4_review_search&vis.ipynb


--------------
**1 리뷰수집**
- 1 crawling Bosch WAJ2416WIN.ipynb
- Amazon 리뷰 데이터를 크롤링하는 코드. Selenium과 BeautifulSoup 사용했음
- 시스템 정책상으로 인해 직접 수집 페이지 설정
   - 1페이지당 약 10개 리뷰
   - WAJ2416WIN 세탁기는 2159개
   - 페이지의 요소가 전부 로딩될 때까지 wait time 설정
      
### _Crawling Data Preprocessing_
  * 크롤링 함수
```python
driver = webdriver.Chrome() # driver 실행

# 저장할 리스트
titles = [] # 타이틀
stars = [] # 별점
dates = [] # 날짜
contents = [] # 컨텐츠

# 수집하고자 하는 리뷰 페이지    
url1 = "write yours" # mine = "[https://www.amazon.in/Bosch-Inverter-Control-Automatic-WAJ2416WIN/dp/B08SR52BSM?th=1](https://www.amazon.in/product-reviews/B08SR52BSM/ref=cm_cr_arp_d_paging_btm_next_2?ie=UTF8&reviewerType=all_reviews&pageNumber=)"
url0 = 0 # 페이지 초기값

start_time = time.time() # 시간 기록

for url2 in range(1, N): # N =  last review page
    url = url1 + str(url2) # 페이지 번호 붙여서 실제 URL 생성
    driver.get(url) # 페이지 요청
    driver.implicitly_wait(10) # 페이지 내 요소 로딩 대기
      # HTML 추출 
    res = driver.page_source
    obj = bs(res,'html.parser') # BeautifulSoup으로 파싱
      # HTML 재추출
    source = driver.page_source
    bs_obj = bs(source, "html.parser") # BeautifulSoup으로 파싱

    for i in bs_obj.findAll('a',{'data-hook':'review-title'}): # review title
        titles.append(i.get_text().strip())
        
    for n in bs_obj.findAll('span',{'data-hook':'review-date'}): # review date
        nn = ''.join(n.get_text().split(' ')[-3:])
        date = datetime.strptime(nn, '%d%B%Y').date()
        dates.append(date)
        
    for a in bs_obj.findAll('span',{'data-hook':'review-body'}): #review body(contents)
        contents.append(a.get_text().strip())
        
    for u in bs_obj.findAll('i', {'data-hook':'review-star-rating'}): #review rating
        stars.append(int(u.get_text()[0]))
        
    print('number of Scrped reviews : ', len(titles))

end_time = time.time()

# 수집된 데이터를 DataFrame으로 정리
df = pd.DataFrame({'Dates':dates,'Ratings':stars, "Titles":titles, "Bodys":contents})

print("소요시간 : {0}".format(round(end_time - start_time)), 2)
```

**2 전처리**
- 2_review_preprocessing.ipynb
- 수집된 리뷰 데이터에 대해 특수문자 제거, 불용어 제거, 표제어 처리 등의 전처리를 수행
- 명사 추출 리스트(`NN`)와 표제어 리스트(`lemmatization`) 등
  
### _Crawling Data Preprocessing_
  * 크롤링 데이터 전처리 함수
```python
def clean_text(texts): 
    corpus = []
    
    for i in tqdm(range(0, len(texts))):
        
        body = texts[i]
        
        body = re.sub('[^a-zA-Z]', ' ', body) # 특수문자 제거 
        body = body.lower().split() # 대문자를 소문자로 변경, 문장을 단어 단위로 구분
        
        df['clean_text'][i] = body
        
        stops = stopwords.words('english')
        stops.append('machine')
        stops.append('product')
        stops.append('bosch')
        
        no_stops = [word for word in body if not word in stops] # 불용어 제거
        df['stopwords_after'][i] = no_stops
        
        tokens_pos = nltk.pos_tag(df['stopwords_after'][i]) # pos tagging (품사 태깅)
        df['pos_tag'][i] = tokens_pos
        
        NN_words = [] # 명사만 추출
        for word, pos in tokens_pos:
            if 'NN' in pos:
                NN_words.append(word)
                df['NN'][i] = NN_words
                
        wlem = nltk.WordNetLemmatizer() # Lemmatization(원형(lemma) 찾기) # nltk에서 제공되는 WordNetLemmatizer을 이용
        lemmatized_words = []
        
        for word in NN_words:
            new_word = wlem.lemmatize(word)
            lemmatized_words.append(new_word)
            df['lemmatization'][i] = lemmatized_words
        
        corpus.append(no_stops) 
        
    return corpus
```

**3 전처리**
- 3_review_LDA.ipynb
- 전처리된 데이터를 기반으로 LDA 토픽 모델링을 수행했음
- 주요 키워드 그룹을 도출하여 고객 요구사항의 패턴을 분석했음
- 
  ![image](https://github.com/user-attachments/assets/22e06ce7-2831-4a01-b532-aa8d4d51de80)


**4 시각화**
- 4_review_search&vis.ipynb
- 주요 키워드 기반 리뷰 검색 기능과 시각화를 구현했음
- 특정 키워드와 관련된 리뷰 추출 및 분석 결과를 시각적으로 표현함

![image](https://github.com/user-attachments/assets/b19ddf2d-7fd0-44a5-8126-d8436cdde0f3)

![image](https://github.com/user-attachments/assets/6ddc85bd-16f9-4b19-8d95-a597c16386a9)

