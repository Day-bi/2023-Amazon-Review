## 데이터 수집

### 수집 대상
- **웹사이트**: [Amazon.com](https://www.amazon.com)
- **제품**: Bosch 세탁기 
- **수집 기간**: 2015년 5월 ~ 2023년 3월

### 수집 방식
- Python 기반 웹 크롤링(Selenium + BeautifulSoup) 사용
- 내부 API, 로그인 등이 필요한 비공개 데이터는 사용하지 않았음
- 수집 항목: 리뷰 제목, 본문, 별점, 날짜

### 예시

| 날짜        | 별점 | 제목                  | 본문         |
|-------------|------|------------------------|--------------|
| 2023-02-20  | 5    | Durable Solid excellent | Superlike    |
| 2023-02-14  | 5    | Great quality          | Fantastic    |
| ...         | ...  | ...                    | ...          |
|2019-11-05|5|Less noise	|Less noise	|
|...|...|...|...|

> 크롤링 코드는 `code/아마존 Bosch WAJ2416WIN 크롤링.ipynb`

---

## 데이터 전처리

- 특수문자 제거, 소문자 변환, 불용어 제거, 표제어 처리 (Lemmatization) 수행
- 품사 태깅 및 명사만 추출하여 분석 대상 단어로 사용
- 전처리 코드는 `code/코드 수정본.py` 또는 `Final_code.ipynb` 내 포함

### _Crawling Data Preprocessing_
  * 크롤링 함수
```python
driver = webdriver.Chrome()

titles = [] # 타이틀
stars = [] # 별점
dates = [] # 날짜
contents = [] # 컨텐츠
    
url1 = "write yours"
url0 = 0 # 페이지 변경

start_time = time.time()

for url2 in range(1,31):
    url = url1 + str(url2)
    driver.get(url)
    driver.implicitly_wait(10) # wait time
    res = driver.page_source
    obj = bs(res,'html.parser')

    source = driver.page_source
    
    bs_obj = bs(source, "html.parser")

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

df2 = pd.DataFrame({'Dates':dates,'Ratings':stars, "Titles":titles, "Bodys":contents})

print("소요시간 : {0}".format(round(end_time - start_time)), 2)
```

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
