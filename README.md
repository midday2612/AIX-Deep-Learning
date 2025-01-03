# AIX-Deep-Learning
aix 팀프로젝트-영화리뷰 감성 분석

## 주제
lstm 모델과 cnn모델을 이용한 영화 리뷰 감성 분류

## Members
      교육학과 김지민
      ICT융합학부 손민지
      전자공학부 윤여준

## 목표(option A)
영화 리뷰에 대한 텍스트를 분석 후 해당 리뷰가 긍정인지, 부정인지 감성분류를 수행한다. 모델 학습 시 lstm 모델과 cnn 모델을 사용한다. 정확도와 학습에 걸리는 시간의 관정에서 두 모델을 비교하고 정확도를 더 높일 수 있는 방법을 고민한다.  

## 동기
자연어 처리는 인공지능 분야에서 중요한 영역 중 하나이다. 텍스트 데이터를 이해하고 처리하는 능력은 다양한 응용 분야에서 핵심 기술로 작용한다. 이 프로젝트를 통해 자연어 처리에 대한 이해를 높이고, 딥러닝 모델을 활용하여 텍스트 감정 분석을 수행함으로써 언어의 의미를 파악하는 방법을 배우고자 한다. CNN과 LSTM은 텍스트 데이터에서 강력한 성능을 보이는 딥러닝 모델 중 하나이다. 이러한 최신 모델을 사용하면 텍스트 데이터의 복잡한 특성을 잘 다룰 수 있다. 프로젝트를 통해 이러한 모델을 어떻게 구현하고 조합하는지를 배울 수 있다. 원본 코드를 참고하여 프로젝트의 목적에 맞게 수정하는 과정에서 코드 리딩하고 적용하는 경험을 한다.
_________________________________________________________________________________________

## 데이터 로드 및 전처리

   1) ### 데이터 로드
      데이터를 다운받는다.  
      Click [here](https://github.com/e9t/nsmc/)  
      네이버영화에서 추출한 **한국어**로 된 영화 리뷰 데이터이다.
      ``` python
      import pickle
      import pandas as pd
      import numpy as np
      import matplotlib.pyplot as plt
      import re
      import urllib.request
      from konlpy.tag import Okt
      from tqdm import tqdm
      from tensorflow.keras.preprocessing.text import Tokenizer
      from tensorflow.keras.preprocessing.sequence import pad_sequences
      ```

      **데이터 소개**  
      각 파일은 id, document, label 세 개의 열로 구성된다.
      - id : 리뷰아이디(네이버에서 제공)
      - document : 실제 리뷰
      - label : 리뷰의 긍정부정 판단 (0:부정, 1: 긍정)
     
      총 리뷰는 다음과 같이 이루어져있다.
      - rating.txt : 20만개의 전체 리뷰 
      - rating_test.txt : 테스트를 위해 보류된 50,000개의 리뷰 (테스트데이터)
      - rating_train.txt : 트레이닝 용 리뷰 150,000개(훈련 데이터)
     
      rating_train.txt(훈련 데이터)와 rating_test.txt(테스트 데이터)를 로드한다. 이때 직접 파일을 가져오는 경우, 데이터 셋이 바뀔 가능성이 있어 로컬 파일에 데이터를 저장한다. 외부데이터에 대한 의존성을 줄여 데이터셋의 안정성을 높인다.

      pandas를 사용해 rating_train.txt는 train_data, rating_test.txt는 test_data에 저장한다.
        ``` python
      train_data = pd.read_table('ratings_train.txt')
      test_data = pd.read_table('ratings_test.txt')
      ```

      train_data의 영화 리뷰 개수를 확인한다.
      
      ``` python
      print('훈련용 리뷰 개수 :',len(train_data)) # 훈련용 리뷰 개수 출력
      ```
      
      ``` python
      훈련용 리뷰 개수 : 150000
      ```
      
      위에서 언급했던 것과 같이 train_data는 총 150,000개의 리뷰로 구성되어 있음을 확인할 수 있다. 데이터의 구성을 확인하기 위해 상위 5개의 샘플을 출력한다.
      
      ``` python
      train_data[:5] # 상위 5개 출력
      ```
      
      |   | id | document | label |
      | :--: | :--: | --: | --: |
      | 0 | 9976970 | 아 더빙... 진짜 짜증나네요 목소리 | 0 |
      | 1 | 3819312 | 흠...포스터보고 초딩영화줄....오버연기조차 가볍지 않구나 | 1 |
      | 2 | 10265843 | 너무재밓었다그래서보는것을추천한다 | 0 |
      | 3 | 9045019 | 교도서 이야기구면 ..솔직히 재미는 없다..평점 조정 | 0 |
      | 4 | 6483659 | 사이몬페그의 익살스런 연기가 돋보였던 영화!스파이더맨에서 늙어보이기만 했던 커스틴 ... | 1 |
      
      표를 보고 중요한 데이터와 그렇지 않은 것을 분류한다.  
      id, document, label 총 3개의 열로 구성된 데이터에서 id는 감성 분류에 의미가 없으므로 무시한다. 리뷰 내용의 document와 긍정(1), 부정(0)을 나타내는 label을 학습하는 모델을 만든다.  
      >추가로 인덱스 2번 샘플에서 띄어쓰기를 하지 않아도 어느정도 말을 이해할 수 있는 한국어 데이터의 특성을 확인할 수 있다.

      train_data 처럼 test_data의 리뷰개수와 상위 5개의 샘플을 확인한다.
       ``` python
      print('테스트용 리뷰 개수 :',len(test_data)) # 테스트용 리뷰 개수 출력
      ```
      ``` python
      테스트용 리뷰 개수 : 50000
      ```
      train_data는 총 50,000개의 리뷰로 구성되어 있다.
      ``` python
      test_data[:5]
      ```

      |   | id | document | label |
      | :--: | :--: | --: | --: |
      | 0 | 6270596 | 굳ㅋ | 1 |
      | 1 | 9274899 | GDNTOPCLASSINTHECLUB | 0 |
      | 2 | 8544678 | 뭐야 이 평점들은....나쁘진 않지만 10점 짜리는 더더욱 아니잖아 | 0 |
      | 3 | 6825595 | 지루하지는 않은데 완전 막장임... 돈주고 보기에는.... | 0 |
      | 4 | 6723715 | 3D만 아니었어도 별 다섯 개 줬을텐데..왜 3D로 나와서 제 심기를 불편하게 하죠?? | 0 |
      
      test_data도 train_data와 동일한 형식으로 이루어져 있는 것을 확인할 수 있다.
      
   3) ### 데이터 정제하기
      데이터를 정제한다. 잘못되거나 불완전한, 관련성이 없거나 중복되고 잘못된 형식으로 된 데이터를 제거하거나 수정한다. 분석에 사용할 수 있도록 데이터를 준비하고 **정제**한다.

      train_data에서 중복된 형식의 데이터가 있는지 확인한다.
      ``` python
      # document 열과 label 열의 중복을 제외한 값의 개수
      train_data['document'].nunique(), train_data['label'].nunique()
      ```
      
      ``` python
      (146182, 2)
      ```  
      결과에서 볼 수 있듯 총 150,000개의 샘플로 구성된 train_data에서 document 열의 중복을 제거한 샘플의 개수가 146,182임을 알 수 있다. 이는 약 4,000개의 중복 샘플이 존재한다는 의미이다.
      label 열은 0또는 1인 오직 두 가지의 값만 가지므로 2가 출력되는 것을 알 수 있다.
      중복 샘플을 제거한다.

      ``` python
      # document 열의 중복 제거
      train_data.drop_duplicates(subset=['document'], inplace=True)
      ```
      
      중복이 제거되었는지 전체 샘플 수를 확인한다.
      
      ``` python
      print('총 샘플의 수 :',len(train_data))
      ```
      ``` python
      총 샘플의 수 : 146183
      ```
      중복 샘플이 제거되었음을 확인할 수 있다.  
      train_data에서 label 값의 긍정, 부정 분포를 확인한다.
      이때, 아나콘다 가상환경에서 기존 코드를 실행시키면 두 개의 그래프가 꼬여서 나오는 문제를 해결하기 위해 plt.show()코드를 추가해 두 개의 그래프를 분리한다.
      ``` python
      train_data['label'].value_counts().plot(kind = 'bar')
      plt.show()
      ```
      ![image](https://github.com/midday2612/aix-deep-learning/assets/149879074/364abe18-b476-49d7-9038-7eac5f87fa28)  
      약 146,000개의 영화 리뷰 샘플 중 그래프 상으로는 긍정과 부정 둘 다 약 72,000개의 샘플이 존재하는 것처럼 보인다. 또한 레이블의 분포도 균일한 것처럼 보인다. 정확하게 몇 개인지 확인한다.
      
      ``` python
      print(train_data.groupby('label').size().reset_index(name = 'count'))
      ```
      
      ``` python
         label  count
      0      0  73342
      1      1  72841
      ```
        
      레이블이 0인 리뷰가 근소하게 많은 것을 확인할 수 있다. 리뷰 중 Null값을 가진 샘플이 있는지 확인한다.  
      
      ``` python
      print(train_data.isnull().values.any())
      ```
      
      ``` python
      True
      ```
      True 값이 나온 것으로 보아 데이터 중에 Null 값을 가진 샘플이 존재한다. 어떤 열에 존재하는지 확인한다.
      
      ``` python
      print(train_data.isnull().sum())
      ```
      
      ``` python
      id          0
      document    1
      label       0
      dtype: int64
      ```
      document 열에서 총 1개가 존재한다. document 열에서 Null값이 존재한다는 것을 조건으로 Null값을 가진 샘플이 어느 인덱스의 위치에 존재하는지 확인한다.
      
      ``` python
      train_data.loc[train_data.document.isnull()]
      ```
      
      |   | id | document | label |
      | :--: | :--: | --: | --: |
      | 25857 | 2172111 | NaN | 1 |  

      출력 결과는 다음과 같다. Null 값을 가진 샘플을 제거한다.
      
      ``` python
      train_data = train_data.dropna(how = 'any') # Null 값이 존재하는 행 제거
      print(train_data.isnull().values.any()) # Null 값이 존재하는지 확인
      ```
      
      ``` python
      False
      ```
      
      성공적으로 제거되었다. 다시 샘플의 개수를 출력하여 1개의 샘플이 제거되었는지 총 데이터 개수를 확인한다.
      ``` python
      print(len(train_data))
      
      ```
      ``` python
      146182
      ```

      이제 데이터의 전처리를 수행해본다. 위의 train_data와 test_data에서 온점(.)이나 ?와 같은 각종 특수문자가 사용된 것을 확인했다. train_data로부터 한글 외의 특수부호를 제거하기 위해서 정규 표현식을 사용한다.
      한글의 정규 표현식은 자음의 경우 'ㄱ ~ ㅎ', 모음의 경우 'ㅏ ~ ㅣ'이다.  
      해당 범위 내에 어떤 자음과 모음이 속하는지 알고 싶다면 아래의 링크를 참고한다.  
      Click [here](https://www.unicode.org/charts/PDF/U3130.pdf)  
      ㄱ ~ ㅎ: 3131 ~ 314E  
      ㅏ ~ ㅣ: 314F ~ 3163
      
      완성형 한글의 범위는 '가 ~ 힣'과 같이 사용한다.
         
      해당 범위 내에 포함된 음절은 아래의 링크에서 확인한다.  
      Click [here](https://www.unicode.org/charts/PDF/UAC00.pdf)
        
      이를 응용해 한글에 속하지 않는 구두점이나 특수문자를 제거한다.  
      위 범위 지정을 모두 반영하여 train_data에 한글과 공백을 제외하고 모두 제거하는 정규 표현식을 수행한다.
      ``` python
      # 한글과 공백을 제외하고 모두 제거
      train_data['document'] = train_data['document'].str.replace("[^ㄱ-ㅎㅏ-ㅣ가-힣 ]","")
      train_data[:5]
      ```
      |   | id | document | label |
      | :--: | :--: | --: | --: |
      | 0 | 9976970 | 아 더빙 진짜 짜증나네요 목소리 | 0 |
      | 1 | 3819312 | 흠 포스터보고 초딩영화줄 오버연기조차 가볍지 않구나 | 1 |
      | 2 | 10265843 | 너무재밓었다그래서보는것을추천한다 | 0 |
      | 3 | 9045019 | 교도서 이야기구면 솔직히 재미는 없다평점 조정 | 0 |
      | 4 | 6483659 | 사이몬페그의 익살스런 연기가 돋보였던 영화스파이더맨에서 늙어보이기만 했던 커스틴 던... | 1 |
      
      상위 5개의 샘플을 다시 출력했다. 정규표현식 수행 후, 기존의 띄어쓰기는 유지되면서 온점이나 구두점 등은 제거되었다.
      네이버 영화 리뷰에서 가져온 데이터 특성 상 영어, 숫자, 특수문자로만 구성된 리뷰를 업로드할 수 있다. 정규표현식이 수행되면 기존의 리뷰는 아무것도 없는 값이 되었을 것이다. train_data에 존재하는 이와 같은 리뷰를 Null값으로 변경해 새로운 Null값이 존재하는지 확인한다.
      ``` python
      train_data['document'] = train_data['document'].str.replace('^ +', "") # white space 데이터를 empty value로 변경
      train_data['document'].replace('', np.nan, inplace=True)
      print(train_data.isnull().sum())
      ```
      ``` python
      id            0
      document    789
      label         0
      dtype: int64
      ```
      Null값이 789개 새로 생겼다, Null값이 있는 행을 5개만 출력한다.
      ``` python
      train_data.loc[train_data.document.isnull()][:5]
      ```
      |   | id | document | label |
      | :--: | :--: | --: | --: |
      | 404 | 4221289 | NaN | 0 |
      | 412 | 9509970 | NaN | 1 |
      | 470 | 10147571 | NaN | 1 |
      | 584 | 71178896 | NaN | 0 |
      | 593 | 6478189 | NaN | 0 |
      
      Null 샘플은 의미가 없으므로 제거한다.
      ``` python
      train_data = train_data.dropna(how = 'any')
      print(len(train_data))
      ```
      ``` python
      145393
      ```
      train_datad의 샘플 개수는 최종적으로 145,393개가 되었다.  
      train_data처럼 test_data 역시 동일한 전처리 과정을 진행한다.
      ``` python
      test_data.drop_duplicates(subset = ['document'], inplace=True) # document 열에서 중복인 내용이 있다면 중복 제거
      test_data['document'] = test_data['document'].str.replace("[^ㄱ-ㅎㅏ-ㅣ가-힣 ]","") # 정규 표현식 수행
      test_data['document'] = test_data['document'].str.replace('^ +', "") # 공백은 empty 값으로 변경
      test_data['document'].replace('', np.nan, inplace=True) # 공백은 Null 값으로 변경
      test_data = test_data.dropna(how='any') # Null 값 제거
      print('전처리 후 테스트용 샘플의 개수 :',len(test_data))
      ```
      ``` python
      전처리 후 테스트용 샘플의 개수 : 48852
      ```
      test_data의 총 개수는 최종적으로 48,852개가 되었다.
      
   5) ### 토큰화
       이제 주어진 텍스트를 더 쉽게 처리, 분석 할 수 있도록 **토큰화**를 진행한다.  우선 그 자체론 실질적인 의미가 존재하지 않아 이후 제거할 불용어를 선정한다.  

      ``` python
      stopwords = ['의','가','이','은','들','는','좀','잘','걍','과','도','를','으로','자','에','와','한','하다']
      ```  
      불용어 선정의 경우 한국어의 조사, 접속사와 같은 일반적인 불용어만을 선택할수도 있지만, 실제론 사용되는 데이터를 검토해 추가, 수정한다.
      이번 프로젝트에선 위의 불용어 만을 사용했다.

      이어서 불용어가 제거된 텍스트에서 각 단어들을 구분해 토큰화를 진행한다.
      여기서 한가지 문제가 생긴다.
      영어의 경우 불용어를 제거한 뒤, 그저 띄어쓰기를 기준으로 단어를 나누면 끝이지만  
      >"Tokenization is an important step in natural language processing." 라는 문장의 토큰화를 진행하면,    
      >>'tokenization', 'important', 'step', 'natural', 'language', 'processing' 와 같이 토큰화된 결과를 볼 수 있다.
      >>>이때 불용어는 'is', 'an', 'in'이다.
      
      한국어의 경우 조사, 어미, 형태소 등이 단어의 의미를 형성하며, 이들은 공백으로만 나누면 제대로 된 의미 파악이 어려울 수 있다.
      따라서 한국어 텍스트를 처리할 때에는 형태소 분석을 통해 단어의 의미 단위를 파악하고, 그 후에 토큰화를 수행하게 된다.  
      해당 과정을 위해 한국어 텍스트의 형태소를 분석해주는 KoNLPy의 Okt를 사용했다.
      ```python
      okt = Okt()
      okt.morphs('와 이런 것도 영화라고 차라리 뮤직비디오를 만드는 게 나을 뻔', stem = True)
      ```
      ```python
      ['오다', '이렇다', '것', '도', '영화', '라고', '차라리', '뮤직비디오', '를', '만들다', '게', '나다', '뻔']
      ```
      Okt는 추가적으로 '이런', '만드는' 과 같은 변형된 형태의 단어도 원형인 '이렇다', '만들다' 변환한다.

      이제 여기서 위에 선정한 불용어를 제거한면 아래의 결과를 얻을 수 있다.
      ```python
      ['오다', '이렇다', '것', '영화', '라고', '차라리', '뮤직비디오', '만들다', '게', '나다', '뻔']
      ```
      실제로 해보니 불용어로 추가해야 할 단어들이 몇몇 보이지만, 우선은 위의 내용대로 설정한 상태에서 학습 데이터를 수정해보겠다.


      ```python
      X_train = []
      for sentence in tqdm(train_data['document']):
          tokenized_sentence = okt.morphs(sentence, stem=True) # 토큰화
          stopwords_removed_sentence = [word for word in tokenized_sentence if not word in stopwords] # 불용어 제거
          X_train.append(stopwords_removed_sentence)
      ```
      훈련에 사용된 데이터셋 X_train의 한국어 텍스트들을 okt를 통해 각각의 단어들로 구분하고 불용어를 제거하여 토큰화를 진행했다.

      결과 확인을 위해 3개의 샘플을 추출 해보았다.
      ```python
      print(X_train[:3])
      ```
      ```pyhton
      [['아', '더빙', '진짜', '짜증나다', '목소리'], ['흠', '포스터', '보고', '초딩', '영화', '줄', '오버', '연기', '조차', '가볍다', '않다'], ['너', '무재', '밓었', '다그', '래서', '보다', '추천', '다']]
      ```
      토큰화가 제대로 진행된 것을 확인했으니, 이제 동일한 작업을 테스트를 위한 X_test에도 진행한다.

      ```python
      X_test = []
      for sentence in tqdm(test_data['document']):
          tokenized_sentence = okt.morphs(sentence, stem=True) # 토큰화
          stopwords_removed_sentence = [word for word in tokenized_sentence if not word in stopwords] # 불용어 제거
          X_test.append(stopwords_removed_sentence)
      ```
      
   6) ### 정수 인코딩
      이제, 토큰화 과정을 거친 단어들을 정수 형태로 인코딩 할 차례다.  
      기계 학습(Machine Leaning)에 사용되는 모델들은 대부분 숫자 형태의 입력을 받는다.  
      그렇기에 텍스트 전처리를 통해 만들어진 단어들을 학습 데이터로 사용하기 위해선 꼭 거쳐야 하는 과정 중 하나라 할 수 있다.  
      ```python
      tokenizer = Tokenizer()
      tokenizer.fit_on_texts(X_train)
      ```
      X_train 데이터에 속한 단어들로 이루어진 거대한 집합을 만들며, 각각의 단어의 하나씩 정수를 할당했다.  
      tokenizer.word_index를 통해 확인 할 수 있다.  
      ```python
      print(tokenizer.word_index)
      ```
      ```python
      {'영화': 1, '보다': 2, '을': 3, '없다': 4, '이다': 5, '있다': 6, '좋다': 7, ... 중략 ... '디케이드': 43751, '수간': 43752}
      ```
      43,000개가 넘는 단어가 있음을 확인했다.
      빈도수가 높은 단어부터 정수를 할당하기 때문에, 낮은 정수가 할당된 단어일수록 더 자주 등장했다는 것을 알 수 있다.  
      반대로 높은 정수의 단어는 그 빈도수가 매우 낮다는 것을 알 수 있다.  
      등장 빈도가 너무 낮은 단어들은 상대적으로 그 영향력이 낮을테니, 원활한 학습을 위해 등장 빈도가 너무 낮은 단어들은 삭제한다.  
      삭제할 단어들을 고르기 위해 등장 빈도가 2회 이하인 단어들의 개수와 전체 리뷰에 해당 단어들이 사용된 비율을 구했보았다.  
      ```python
      threshold = 3
      total_cnt = len(tokenizer.word_index) 
      rare_cnt = 0 
      total_freq = 0 
      rare_freq = 0 

      for key, value in tokenizer.word_counts.items():
          total_freq = total_freq + value

          if(value < threshold):
              rare_cnt = rare_cnt + 1
              rare_freq = rare_freq + value

      print('단어 집합(vocabulary)의 크기 :',total_cnt)
      print('등장 빈도가 %s번 이하인 희귀 단어의 수: %s'%(threshold - 1, rare_cnt))
      print("단어 집합에서 희귀 단어의 비율:", (rare_cnt / total_cnt)*100)
      print("전체 등장 빈도에서 희귀 단어 등장 빈도 비율:", (rare_freq / total_freq)*100)
      ```
      ```python
      단어 집합(vocabulary)의 크기 : 49586
      등장 빈도가 2번 이하인 희귀 단어의 수: 28787
      단어 집합에서 희귀 단어의 비율: 58.054692856854764
      전체 등장 빈도에서 희귀 단어 등장 빈도 비율: 1.909214704259857
      ```
      등장 빈도가 2회 이하인 단어들의 갯수는 전체 단어 집합의 절반을 넘지만, 실제 리뷰에 사용된 비율은 2퍼센트가 되지 않는 것으로 보아 삭제해도 괜찮다고 판단했다.


      단어 집합의 크기를 등장 빈도가 3회 이상인 단어들의 수로 제한하고, x_train에 tokenizer를 적용하면 등장 빈도가 3회 이상인 단어들만 남는다.
      ```python
      vocab_size = total_cnt - rare_cnt + 1 #rare_cnt = 등장 빈도가 2회 이하인 단어들의 갯수
      print('단어 집합의 크기 :',vocab_size) #등장 빈도가 3회 이상인 단어의 갯수로 vocab_size 수정
      ```
      ```python
      단어 집합의 크기 : 20800
      ```
      이렇게 수정한 vacab_size를 이용해 다시 단어를 정수로 인코딩한다.
      ```python
      tokenizer = Tokenizer(vocab_size) 
      tokenizer.fit_on_texts(X_train)
      X_train = tokenizer.texts_to_sequences(X_train)
      X_test = tokenizer.texts_to_sequences(X_test)
      ```
      정수 인코딩이 제대로 진행되었는지 확인하기 위해 다시 3개의 샘플을 뽑아 확인한다.
      ```python
      print(X_train[:3])
      ```
      ```python
      [[50, 454, 16, 260, 659], [933, 457, 41, 602, 1, 214, 1449, 24, 961, 675, 19], [386, 2444, 2315, 5671, 2, 222, 9]]
      ```
      정수가 제대로 할당된 것을 확인 할 수 있다.

      추가로 train_data와 test_data에서 label을 뽑아 y_train과 y_test를 만들었다.
      ```python
      y_train = np.array(train_data['label'])
      y_test = np.array(test_data['label'])
      ```

      
   7) ### 빈 샘플 제거

      4번까지의 과정을 통해  빈도수가 낮은 단어들을 삭제했고, 그 결과 몇몇 샘플들이 완전히 비어있게 되었다. 빈 샘플을 제거하여 모델이 효과적으로 학습하고 일반화할 수 있도록 도와주며, 불필요한 계산을 줄여 효율성을 높일 수 있다. 
      ```python
      drop_train = [index for index, sentence in enumerate(X_train) if len(sentence) < 1]
      ```

      각 샘플들의 길이를 확인하면서 길이가 0인 샘플들을 찾아내고 인덱스(위치)를 추출한다. drop_train에는 빈 샘플들의 인덱스가 저장된다. 
       ```python
       # 빈 샘플들을 제거
       X_train = np.delete(X_train, drop_train, axis=0)
       y_train = np.delete(y_train, drop_train, axis=0)
       print(len(X_train))
       print(len(y_train))
      ```
       <img width="43" alt="image" src="https://github.com/midday2612/aix-deep-learning/assets/109676875/fa5639af-3603-4257-8909-06017e6567f9">

      빈 샘플을 제거하는 코드이다. 훈련데이터 샘플의 개수가 145881개로 줄어들었다. 빈 샘플이 잘 제거되었음을 확인할 수 있다. 
       
   8) ### 패딩
      패딩은 시퀀스 데이터의 길이를 맞춰주는 작업이다. 텍스트 데이터들의 길이가 각각 다른경우 모델이 효과적으로 학습하기 위해서는 패딩을 통해 입력 데이터의 길이를 일정하게 맞춰줄 수 있다.

      일반적으로는 가장 긴 시퀀스의 길이에 맞춰 패딩을 진행하지만 해당 데이터 셋에서는 데이터 셋 간의 길이 차이가 크기 때문에 리소스 효율성을 고려하여 적절한 패딩 길이를 찾을 필요가 있다.

      ```python
      print('리뷰의 최대 길이 :',max(len(review) for review in X_train))
      print('리뷰의 평균 길이 :',sum(map(len, X_train))/len(X_train))
      plt.hist([len(review) for review in X_train], bins=50)
      plt.xlabel('length of samples')
      plt.ylabel('number of samples')      
      plt.show()
      ```
      코드를 실행시켜 보면 다음과 같은 그래프를 얻을 수 있다.
      <img width="477" alt="image" src="https://github.com/midday2612/aix-deep-learning/assets/109676875/5726da5e-1d70-4c3d-a25d-a8a9cf760f79">


      가장 긴 리뷰의 길이는 95이다. 대다수의 샘플이 잘리지 않도록 적절한 max_len값을 선택해야 하는데, 아래의 코드를 실행하여 전체 샘플 중에서 길이가 max_len 이하인 샘플의 비율을 계산하여 얼마나 많은 샘플이 해당 길이 이하로 잘리지 않을지 확인한다.

      max_len의 값을 바꿔보며 테스트한 결과 최종적으로 max_len = 35으로 설정하였다. 전체 샘플중 길이가 max_len이하인 샘플 비율이 95에 가까운 값으로 선정했다.  


      ```python
      max_len = 35
      below_threshold_len(max_len, X_train)
      ```
      이 코드를 실행 시키면 전체 샘플 중 길이가 35 이하인 샘플의 비율이  94.71624131997999임을 확인할 수 있다. 
      <img width="432" alt="image" src="https://github.com/midday2612/aix-deep-learning/assets/109676875/201cd2f0-ccf8-4d07-9e27-a83ba202ac75">


      ```python
      X_train = pad_sequences(X_train, maxlen=max_len)
      X_test = pad_sequences(X_test, maxlen=max_len)
      ```
      마지막으로 패딩을 수행한다. 
      pad_sequences 함수는 텐서플로(TensorFlow) 라이브러리의 keras.preprocessing.sequence 모듈에 있다. 
   
   9) ### 전처리 결과 저장
       동일한 cnn모델과 lstm 모델을 정확하게 비교하기 위해 같은 전처리 데이터를 이용해야 한다. 따라서 전처리 결과를 텍스트 파일로 저장하였다.

      <img width="693" alt="image" src="https://github.com/midday2612/aix-deep-learning/assets/109676875/9f8248a6-4e04-49d7-9e21-a5ada52bd371">
      전처리 결과가 저장되었다.


      ![Uploading image.png…]()
      lstm과 cnn 모델 학습 시 사용되는 변수들도 따로 텍스트 파일에 저장하였다. 

      
_________________________________________________________________________________________

## LSTM을 이용한 예측 모델
1) **LSTM 이란?**
   > LSTM은 RNN의 한 종류로, RNN의 장기 의존성 문제(long-term dependencies)를 해결하기 위해서 나온 모델이다. 따라서 직전 데이터 뿐만 아니라, 좀더 거시적으로 과거 데이터를 고려하여 미래 데이터를 예측하기 위해 나온 모델이다.
![image](https://github.com/midday2612/aix-deep-learning/assets/149879074/8c015b35-57aa-4dd3-80e3-d8b1c1cd72e7)
   LSTM은 체인 구조를 가지고 있지만, 4개의 Layer가 특별한 방식으로 서로 정보를 주고 받도록 되어 있다.

2) **LSTM 코드 설명 및 테스트 정확도 측정**
   - LSTM 모델 설명  
     하이퍼파라미터인 임베딩 벡터의 차원은 100, 은닉 상태의 크기는 128이다. 모델은 다 대 일 구조의 LSTM을 사용한다. 해당 모델은 마지막 시점에서 두 개의 선택지 중 하나를 예측하는 이진 분류 문제를 수행하는 모델이다.
     이진 분류 문제의 경우, 출력층에 로지스틱 회귀를 사용해야 하므로 활성화 함수로는 시그모이드 함수를 사용하고, 손실 함수로는 크로스 엔트로피 함수를 사용한다. 하이퍼파라미터인 배치 크기는 64이며, 15에포크를 수행한다.
   - 코드 설명  
     + EarlyStopping(monitor='val_loss', mode='min', verbose=1, patience=4) : 검증 데이터 손실(val_loss)이 증가하면, 과적합 징후므로 검증 데이터 손실이 4회 증가하면 정해진 에포크가 도달하지 못하였더라도 학습을 조기 종료(Early Stopping)한다  
     + ModelCheckpoint : 검증 데이터의 정확도(val_acc)가 이전보다 좋아질 경우에만 모델을 저장  
     + validation_split=0.2 : 훈련 데이터의 20%를 검증 데이터로 분리해서 사용하고, 검증 데이터를 통해서 훈련이 적절히 되고 있는지 확인. 검증 데이터는 기계가 훈련 데이터에 과적합되고 있지는 않은지 확인하기 위한 용도로 사용.
     ```python
     from tensorflow.keras.layers import Embedding, Dense, LSTM
     from tensorflow.keras.models import Sequential
     from tensorflow.keras.models import load_model
     from tensorflow.keras.callbacks import EarlyStopping, ModelCheckpoint

     embedding_dim = 100
     hidden_units = 128

     model = Sequential()
     model.add(Embedding(vocab_size, embedding_dim))
     model.add(LSTM(hidden_units))
     model.add(Dense(1, activation='sigmoid'))

     es = EarlyStopping(monitor='val_loss', mode='min', verbose=1, patience=4)
     mc = ModelCheckpoint('best_model.h5', monitor='val_acc', mode='max', verbose=1, save_best_only=True)

     model.compile(optimizer='rmsprop', loss='binary_crossentropy', metrics=['acc'])
     history = model.fit(X_train, y_train, epochs=15, callbacks=[es, mc], batch_size=64, validation_split=0.2)
     ```  
      
     코드를 진행할 시 조기 종료 조건에 따라서 9 에포크에서 훈련이 멈춘다. 훈련이 종료 될 시, 테스트 데이터에 대해서 정확도를 측정한다. 훈련과정에서 검증 데이터가 가장 높았을 때 저장된 모델인 'best_model.h5'를 로드한다.  
      
     ```python
     loaded_model = load_model('best_model.h5')
     print("\n 테스트 정확도: %.4f" % (loaded_model.evaluate(X_test, y_test)[1]))
     ```
     ```python
     테스트 정확도: 0.8544
     ```  
     테스트 정확도는 85.44%이다.
     모델과 마찬가지로 토크나이저도 다음과 같이 파일로 저장 후 다시 로드한다.
     ```python
     with open('tokenizer.pickle', 'wb') as handle:
     pickle.dump(tokenizer, handle)

     with open('tokenizer.pickle', 'rb') as handle:
     tokenizer = pickle.load(handle)
     ```
     
3) **리뷰 예측하기**  
      임의의 리뷰에 대해서 예측하는 함수를 만들어보자. model.predict()를 사용하여 현재 학습한 model에 새로운 입력에 대해서 예측값을 얻는다. 이때 model.fit()을 할 때와 마찬가지로 새로운 입력에 대해서도 동일한 전처리를 수행 후에 model.predict()의 입력으로 사용해야 한다.
      ```python
      def sentiment_predict(new_sentence):
      new_sentence = re.sub(r'[^ㄱ-ㅎㅏ-ㅣ가-힣 ]','', new_sentence)
      new_sentence = okt.morphs(new_sentence, stem=True) # 토큰화
      new_sentence = [word for word in new_sentence if not word in stopwords] # 불용어 제거
      encoded = tokenizer.texts_to_sequences([new_sentence]) # 정수 인코딩
      pad_new = pad_sequences(encoded, maxlen = max_len) # 패딩
      score = float(loaded_model.predict(pad_new)) # 예측
      if(score > 0.5):
            print("{:.2f}% 확률로 긍정 리뷰입니다.\n".format(score * 100))
      else:
            print("{:.2f}% 확률로 부정 리뷰입니다.\n".format((1 - score) * 100))
      ```
      ```python
      sentiment_predict('이 영화 개꿀잼 ㅋㅋㅋ')
      ```
      ```python
      97.76% 확률로 긍정 리뷰입니다.
      ```
      ```python
      sentiment_predict('이 영화 핵노잼 ㅠㅠ')
      ```
      ```python
      98.55% 확률로 부정 리뷰입니다.
      ```
      ```python
      sentiment_predict('이딴게 영화냐 ㅉㅉ')
      ```
      ```python
      99.91% 확률로 부정 리뷰입니다.
      ```
      ```python
      sentiment_predict('감독 뭐하는 놈이냐?')
      ```
      ```python
      98.21% 확률로 부정 리뷰입니다.
      ```
      ```python
      sentiment_predict('와 개쩐다 정말 세계관 최강자들의 영화다')
      ```
      ```python
      80.77% 확률로 긍정 리뷰입니다.
      ```
_________________________________________________________________________________________

## CNN을 이용한 예측 모델
1) **데이터 전처리**  
   앞서 '데이터 로드 및 데이터 전처리' 파트에서 로드 및 전처리한 데이터를 사용한다.

2) **Multi-Kernel 1D CNN Model 만들기**  
   커널 사이즈가 3, 4, 5인 3 종류의 커널을 각각 128개씩 사용하는 영화리뷰 감성을 판별하는 CNN 모델을 만들어보자.    

    
   먼저, 필요한 tool들을 임포트한다.
    ```python
    from tensorflow.keras.models import Sequential, Model
    from tensorflow.keras.layers import Embedding, Dropout, Conv1D, GlobalMaxPooling1D, Dense, Input, Flatten, Concatenate
    from tensorflow.keras.callbacks import EarlyStopping, ModelCheckpoint
    from tensorflow.keras.models import load_model
    ```

   128차원의 임베딩 벡터(하이퍼 파라미터),  
   0.5, 0.8의 두 가지 드롭아웃 레이트,  
   3종류의 커널을 각각 128개씩 사용한다.
    ```python
    embedding_dim = 128
    dropout_ratio = (0.5, 0.8)
    num_filters = 128
    hidden_units = 128
     ```

   CNN모델의 입력과 임베딩 레이어
   ```python
   model_input = Input(shape = (max_len,))
      z = Embedding(vocab_size, embedding_dim, input_length = max_len, name="embedding")(model_input)
      z = Dropout(dropout_ratio[0])(z)
   ```
   임베딩 층 이후에 드롭아웃 비율이 0.5인 드롭아웃 레이어를 위치시켜 50퍼센트 드롭아웃을 진행했다.

   이어서 3X3, 4X4, 5X5인 3 종류의 커널을 각각 128개씩 사용하는 합성곱 레이어와 맥스풀링을 진행하는 맥스풀링층과 각각의 커널에 대한 결과들을 하나로 연결하는 Concatenate 레이어, 그리고 드롭아웃(비율 0.8)을 진행하는 드롭아웃 레이어를 만들었다.
   ```python
   conv_blocks = []

      for sz in [3, 4, 5]:
          conv = Conv1D(filters = num_filters,
                               kernel_size = sz,
                               padding = "valid",
                               activation = "relu",
                               strides = 1)(z)
          conv = GlobalMaxPooling1D()(conv)
          conv_blocks.append(conv)

      z = Concatenate()(conv_blocks) if len(conv_blocks) > 1 else conv_blocks[0]
      z = Dropout(dropout_ratio[1])(z)
   ```


   Fully Connected(완전연결) 레이어를 추가하고, 최종적으로 활성화 함수로 sigmoid를 사용해 1과 0(긍정과 부정)을       반환하는 노드 1개의 레이어 위치시시키고 모델을 컴파일한다.
      ```python
      z = Dense(hidden_units, activation="relu")(z)
      model_output = Dense(1, activation="sigmoid")(z)
      model = Model(model_input, model_output)
      model.compile(loss="binary_crossentropy", optimizer="adam", metrics=["acc"])
      ```
      
     이걸로 CNN 모델의 설계가 끝이났다.
     
3) **모델 학습과 평가**
   이제 모델을 학습하고 테스트 데이터를 통해 모델의 성능을 평가할 차례다.
   
   ```python
   es = EarlyStopping(monitor='val_loss', mode='min', verbose=1, patience=4)
   mc = ModelCheckpoint('CNN_model.h5', monitor='val_acc', mode='max', verbose=1, save_best_only=True)
   ```
   모델 학습을 진행하기 전에, 조기 종료와 체크포인트를 통한 모델 성는이 뛰어난 모델을 저장해야한다.

   조기 종료는 모델 학습이 진행되는 동안 val_loss가 일정 epoch가 지나도 개선(loss가 줄어드는 것)되지 않으면 모델이 과적합 되어간다고 판단하여 미리 학습이 종료되는 것을 이야기한다.

       
   모델이 임의의 데이터들에 대해 모두 균일한 성능을 보여주는 것이 아닌, 오로지 train_data에 한하여 좋은 성능을 보여주는 것을 과적합이라한다.   
   이러한 경우, 검증을 위해 val_loss를 모니터링 해가며 모델이 과적합 되기 이전에 학습을 종료한다.  
   추가적으로 조기 종료는 모델의 성능이 개선되지 않을 때 무의미한 학습 과정을 정지하여 자원과 시간을 절약하는데도 도움이 된다.

     아래의 코드는 val_loss가 4epoch 동안 개선되지 않으면 모델 학습을 조기종료하는 코드이다.
    ```python
   es = EarlyStopping(monitor='val_loss', mode='min', verbose=1, patience=4)
   ```

      
   추가로 학습이 종료되기 전까지 가장 성능이 좋았던(val_acc가 가장 큰) 모델을 추후에 사용하기 위해 모델의 가중치를 저장하는 것이 체크포인트를 통한 모델 저장이다.

    val_acc가 가장 큰 모델을 체크포인트로 저장한다.
   ```python
   es = EarlyStopping(monitor='val_loss', mode='min', verbose=1, patience=4)
   mc = ModelCheckpoint('CNN_model.h5', monitor='val_acc', mode='max', verbose=1, save_best_only=True)
   ```

   모델 학습이 끝났으니, 체크포인트를 통해 저장한 모델을 로드하여 테스트 데이터에 대해 평가한다.
   ```python
   loaded_model = load_model('CNN_model.h5')
   print("\n 테스트 정확도: %.4f" % (loaded_model.evaluate(X_test, y_test)[1]))
   ```
   ```python
   테스트 정확도: 0.8479
   ```

   테스트 데이터에 대해 84퍼센트의 정확도를 보이는 모델이 완성되었다.  
   

5) **리뷰 예측**  
   lstm 모델에서와 동일하게 sentiment_predict 예측 함수를 이용해 리뷰를 예측한다.
   ```python
   sentiment_predict('이 영화 개꿀잼 ㅋㅋㅋ')
   ```
   ```python
   93.73% 확률로 긍정 리뷰입니다.
   ```
   실제 최근 영화의 네이버 영화 리뷰를 가져와 예측해보았다.
    ```python
   sentiment_predict('진짜 그냥 말이 필요없고 모두가 봐야할 영화.오랜만에 영화다운 영화를 본것같았고권선징악을 바라고 빌면서 보게되지만 결과가 이미 나와있어서 더 열받게 함.너무 뜻깊은 영화이고 황정민 연기…진짜 개 킹받아ㅠㅜ')
   ```
   ```python
   88.56% 확률로 긍정 리뷰입니다.
   ```

   
## LSTM과 CNN의 성능 비교
1. 실행 환경

   AMD 라이젠 5 5600 H cpu,  16g 메모리, 윈도우 환경
     
2. 실행 시간

   LSTM 모델: 630.4515 초
   CNN 모델: 589.4065 초
      
3. 정확도

   LSTM 모델 정확도: 0.8591
   CNN 모델 정확도: 0.8465
            
4. 결과 분석 

      두 모델 모두 에폭을 10으로 조정하고 학습시켰다. LSTM 모델이 CNN 모델보다 약 41.045 초 더 소요되었다. 모델의 학습 및 예측 속도에 대한 추가 최적화가 필요할 수 있다.LSTM 모델이 조금 더 높은 정확도를 보이고 있으며, 따라서 현재 상황에서는 더 성능이 좋은 모델로 판단된다.모델의 학습 속도를 향상시키기 위해 하이퍼파라미터를 바꿔가며 테스트 하고 데이터 증강 등의 방법을 통해 모델의 과적합 문제를 해결해나갈 수 있다. 

## 실험 전 예측과 실제 결과 비교    
  LSTM과 같이 자기 회귀성을 가진 모델들은 데이터의 순서가 정보를 내포하고 있는 '순차 데이터'들에 대한 성능이 그 외의 (DNN, CNN) 모델들에 비해 성능이 좋을 것이라 예측했다.  
  하지만, 실제로 실험해본 결과 LSTM 모델의 예측 성능이 CNN 모델보다 좋긴 했으나 두드러지는 차이가 보이진 않았다.  
  오히려 문장 속 단어를 1개씩 입력 받아 학습을 진행하는 구조 탓에 모델을 훈련 시키는데 7%가량의 시간이 더 소모되었다.    
  왜 예상과 결과가 달랐을까.  
  우선 LSTM의 가장 큰 장점은 현재 입력으로 사용하는 단어와 순서상 멀리 떨어진 단어라 할 지라도 그 연관성을 파악할 수 있다. 그렇기에 긴 문장이라 할지라도 그 의미를 파악하고 예측에 사용할 수 있다.  
  반면 DNN의 경우 단어들 각각을 동시에 입력 데이터로 사용해 무엇이 먼저 나왔고, 무엇이 문장의 뒤에 위치하는지 파악할 수 없다. 즉, 단어가 아닌 문장의 의미는 파악할 수 없기에 자연어 처리에 좋지 않은 성능을 보인다.  
  그렇다면 우리가 LSTM과 비교하기 위해 사용한 CNN의 경우는 어떨까.  
  우선 CNN의 C를 맡고 있는 Convoultion은 중심 데이터와 그 주변 데이터의 관계를 학습할 수 있다.  
  이말은 곧 중심이 되는 단어 주변의 다른 단어들과의 관계를 모델이 학습할 수 있다는 말이다. 그렇기에 LSTM만큼 문장 전체의 의미를 파악하진 못했더라도 간단한 영화 리뷰 글에 한해선 모델이 문장의 의미를 대략적으로 파악 할 수 있었던 것으로 보인다.  

  결론적으로 간단한 구조의 모델과, 너무 길지 않은 문장에선 LSTM과 CNN 모델의 예측 정확도에 큰 차이가 나지 않고, 오히려 문장 속 단어 전부를 한번에 입력으로 받는 CNN이 학습 속도 면에서 더 이점을 갖게 된다는 것을 알 수 있었다.

## 역할 분담
|이름|역할|
|---|---|
|김지민|데이터로드 및 전처리/ LSTM을 이용한 예측 모델/비디오|
|손민지|코드 수정/데이터 전처리 part5, 6/lstm,cnn 모델 비교|
|윤여준|part 3,4 / cnn을 이용한 예측 모델|

## 느낀점
-김지민

데이터를 로드하는 방법과 전처리하는 방법, 그리고 코딩을 하는 방법에 대해 배울 수 있었다. 딥러닝은 처음 해보는 것이어서 어려웠지만, 팀원분들이 많이 도와주셔서 해낼 수 있었다. LSTM과 CNN모델을 비교하면서 LSTM이뭐고 CNN이 뭔지 배울 수 있었다. 또 둘을 비교하면서 다음에 딥러닝 코드를 짠다면 어떤 모델을 사용하면 좋을지 생각해 보는 기회가 되었다.  

-손민지


자연어 처리에서 데이터를 어떻게 전처리하는지 배울 수 있었다. 정확도를 위해서는 모델 튜닝도 중요하지만 전처리 과정에서 불필요한 데이터를 제거하는 과정이 필수적이라는 것을 알 수 있었다. 모델 학습 과정에서 lstm과 cnn 모델의 동작구조를 이해할 수 있었다. 하이퍼파라미터를 조정하며 모델의 정확도를 높여보고 싶다. 

-윤여준

문장에는 단어 하나하나의 정보 뿐만이 아니라 '어순'이라는 순차 정보 또한 담겨있다.   
때문에 자연성 처리에선 일반적인 DNN과는 다른, 단어의 순서마저 학습에 사용하는 RNN 스타일의 모델(해당 글에선 LSTM)이 사용된다는 점을 배울 수 있는 기회였다.  
또한 CNN은 2차원 데이터만을 다루는 모델로 알고 있었지만, 1차원 데이터에서도 사용될 수 있다는 점도 알게 되었다.    
  이번 프로젝트는 사용되는 데이터의 형태, 목적에 맞게 모델의 구조를 결정하는 능력을 키울 수 있는 좋은 기회였던 것 같다.

## 참고 자료 및 사이트
-lstm 감성분류
https://wikidocs.net/44249

-cnn 감성분류
https://wikidocs.net/85337

## 블로그 소개 영상
https://youtu.be/3nUEBSZHC2M?si=fWBBSXm4jn2cs4uy
