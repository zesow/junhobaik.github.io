# 개요

 방학 프로젝트로 댓글 분석을 하고 있다. 나는 데이터 엔지니어링 파트를 맡아서, 데이터 수집 및 저장하는 파이프라인을 자동화 하고 있다. 파이프라인 중 첫 번째에 해당하는 데이터 수집 부분에서 실습한 내용이다.



# 데이터 수집

 이번 프로젝트 같은 경우에는 데이터 수집 대상이 댓글이다. 그러나 내가 원하는 사이트에서는 api를 제공하지 않기 때문에, 그 쪽 서버에 무리가 가지 않는 선에서 데이터를 크롤링 하기로 결정했다.



## 뽑아낼 정보 정의하기

 직접 쇼핑몰에 들어가서 리뷰들을 확인하며, 도움이 되겠다 싶은 정보는 일단 다 뽑아내기로 했다.

그래서 최종적으로 뽑아낸 정보들은 다음과 같다.

- pTitle : 상품명
- cId :고객아이디
- rNo : 리뷰번호
- _id : 쇼핑몰번호 + 리뷰번호. mongodb의 key로 쓰인다.
- rDate : 리뷰 작성일
- helpful : 리뷰 추천 수
- rScore : 상품 평가한 점수
- height : 고객키
- weight : 고객몸무게
- bodysize : 고객사이즈
- pColor : 상품색
- pSize : 상품사이즈
- pURL :상품 url
- PID : 상품 id
- desc : 리뷰내용



## 뽑아내기 - python의 beautifulsoup4 를 사용

정보를 정의하고 직접 크롤링에 들어갔다.

크롤링 툴로 가장 유명한 뷰티플수프를 이용해서 했다.

 필요한 부분만 그때그때 찾아가면서 하니 나중에는 html이 다 비슷해서 대충 이건 이렇게 뽑으면 되겠다는 감이 왔다.

```python
from urllib.request import urlopen
from bs4 import BeautifulSoup
import json
from collections import OrderedDict
from urllib.error import URLError

page = 1

count = 1
jsonList = []
folder_name = 'read_add'
while True:
    url = urlbefore + str(page) + urlafter
    try:
        html = urlopen(url).read()
    except URLError as e:
        print(e)
        continue

    bsObj = BeautifulSoup(html,"html.parser")

    rObject = bsObj.findAll("li",{"class":"reviews_index_list_review"})

    for obj in rObject:
        options = obj.find("div",{"class":"review_options"})

        reviews = obj.find("div",{"class":"reviews_index_list_review__message_expanded"})
        reviewers = obj.find("span",{"class":"reviews_index_list_review__name"})
        evals = obj.find("span",{"class":"reviews_index_list_review__rating_item reviews_index_list_review__text_rating"})
        products = obj.find("a",{"class":"reviews_index_list_review__title_text js-link-iframe"})
        helpfuls_plus = obj.find("strong",{"class":"js-like-score-plus"})
        helpfuls_total = obj.find("strong",{"class":"js-like-score-total"})
        images = obj.findAll("img",{"class":"js-review-image smooth"})
        review_json = OrderedDict()
        try:
            review_json['pTitle'] = products.get_text().strip()
            review_json['cId'] = reviewers.get_text().strip()
            review_json['rNo'] = int(obj.get('id').split('_')[1])
            review_json['_id'] = 'stylenanda_'+str(review_json['rNo'])
            review_json['helpful'] = [int(helpfuls_plus.get_text().strip()),int(helpfuls_total.get_text().strip())]
            rScore_temp = evals.get_text().replace('-','').strip()
            if(rScore_temp == "아주 좋아요"):
                review_json['rScore'] = 5.0
            elif(rScore_temp == "맘에 들어요"):
                review_json['rScore'] = 4.0
            elif(rScore_temp == "보통이에요"):
                review_json['rScore'] = 3.0
            elif(rScore_temp == "그냥 그래요"):
                review_json['rScore'] = 2.0
            else:
                review_json['rScore'] = 1.0

            if options is not None:
                titles = options.findAll("div",{"class":"review_option__title"})
                contents = options.findAll("div",{"class":"review_option__content"})


                for j in range(len(titles)):
                    if(titles[j].get_text() == "선택한 옵션"):
                        continue

                    if(titles[j].get_text() == 'HEIGHT'):
                        review_json['height'] = int(contents[j].get_text().split(' ')[0])
                    elif(titles[j].get_text() == 'SHOE SIZE'):
                        review_json['shoeSize'] = int(contents[j].get_text().split(' ')[0])
                    elif(titles[j].get_text() == 'WEIGHT'):
                        review_json['weight'] = contents[j].get_text()
                    elif(titles[j].get_text() == 'BODY SIZE'):
                        review_json['bodySize'] = contents[j].get_text()

                chooses_titles = options.findAll("span",{"class":"review_option__product_option_key"})
                chooses_contents = options.findAll("span",{"class":"review_option__product_option_value"})
                for j in range(len(chooses_titles)):
                    if(chooses_titles[j].get_text() == '색상:'):
                        review_json['pColor'] = chooses_contents[j].get_text()
                    else:
                        review_json['pSize'] = chooses_contents[j].get_text()
            review_json['pURL'] = products.get('data-url')
            review_json['pID'] = review_json['pURL'].split('product_no=')[1]

            if len(images) is not 0:
                imgList=[]
                for j in images:
                    imgList.append(j.get('src').replace("//",""))
                review_json['rImages'] = imgList

		

            review_json['desc'] = reviews.get_text().strip()
        except:
            continue
        json_now = json.dumps(review_json, ensure_ascii=False, indent="\t")
		#print(json_now)
        jsonList.append(review_json)

    # print(jsonList)
    if len(jsonList) == 0:
        break
    if page % 250 == 0 :
        fname = folder_name + '/stylenanda_'+str(count)+'.json'
        with open(fname, 'w') as fp:
            json.dump(jsonList, fp, ensure_ascii=False, indent="\t")

        count += 1
        jsonList = []

    page = page + 1
```



전체 코드에서 url 부분만 삭제하였다.

애먹은 포인트가 몇 가지 있는데, 처음에 크롤링 코드를 짜고 json 파일을 만들 때 어떻게 jsonArray형식으로 만들 건지 고민했다. 단순히 list 자료형에 append를 이용해서 json 파일을 집어넣다가, 일정 갯수마다 json.dump를 이용하면 손쉽게 만들어 졌다.

 그리고 크롤링의 특성상 돌려 놓고 오류가 나든 성공이 나든 오랜 시간을 기다려야 하는데, 중간에 오류가 나면 처음부터 다시 해야하는게 어려웠다. 그래서 아얘 코드 전체를 try로 묶어놓고 모든 에러에 대해서 에러가 나면 해당 리뷰는 건너뛰도록 처리해 버렸다.



이렇게 하고, nohup이라는 리눅스 데몬 실행 프로그램을 이용해서 백그라운드에서 돌려주었다.



# 후기

 이렇게 해서 한 번 데이터 수집을 끝마쳐 보았다. 이제 이 프로그램을 발전시켜, 하루에 한 번씩 자동으로 새로운 리뷰를 크롤링하여 mongodb에 집어넣도록 최종 파이프라인을 완성할 생각이다.