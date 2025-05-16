
리뷰 수집, 전처리, 분석 코드
===============
  
**주요 파일 목록**
--------------
**리뷰수집**
- 아마존 Bosch WAJ2416WIN 크롤링.ipynb  
  Amazon 리뷰 데이터를 크롤링하는 코드. Selenium과 BeautifulSoup 사용했음.
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

**전처리**
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
**실행 순서**
---------

1. 아마존 Bosch WAJ2416WIN 크롤링 → 리뷰 데이터 수집
2. Final_code.ipynb 실행 → 리뷰 데이터 전처리 및 분석
