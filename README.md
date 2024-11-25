# Transformer-(Multiclass)Classifier 모델 설명

0. 구동환경

   파이썬 버전 : 3.8.20

   딥러닝 프레임워크 : pytorch 1.12.1

   GPU : RTX A6000 * 2

   CUDA : Cudatoolkit 11.3, cudNN 8.1.0.77

1. 모델구조 (trasnformer-classifier7.ipynb)
   
   1501 -> 1501*8 -> Transformer Encoder -> 4  

   input feature 가 텍스트가 아닌 유전자의 위암관련 위치 1501곳의 mRNA 발현량 0~31 사이의 실수이기 때문에

   이 feature 값들의 정수부분만 잘라 이를 원 트랜스포머의 단어를 토큰화 한 결과처럼 사용하였는데,

   그 가지 수를 32개가 아닌 320개로 늘리기 위해 Min-Max Normalization 후 319를 곱하였음.

   임베딩벡터차원은 32000 단어일 때 512였던 것을 감안, 비례대로라면 5를 넣어야 하나 작동최소단위인 8로 함.  

   이렇게 늘어난 1501 * 8 행렬 데이터에 position embedding 처리를 해 주고

   트랜스포머 인코더를 통과시킨다음 Node 개수가 4개인 FC Layer 를 통과시켰으며,

   마지막에 torch.log_softmax를 달아주어 훈련시 NLL Loss 에서 가장 큰 값의 인덱스를 뽑게 함. 

2. 훈련데이터

   Dataset : Train 325, Test 82

   X_Train : (325, 1501) torch.Int32 Tensor (0~319사이의 정수)

   Y_Train : (325,) torch.Int64 Tensor (0,1,2,3 중 하나)

   X_Test : (82, 1501) torch.Int32 Tensor (0~319사이의 정수)

   Y_Test : (82,) torch.Int64 Tensor (0,1,2,3 중 하나)

   Batch_size : 10 (장착된 GPU에 따라 달라져도 됨)

   Optimizer : Adam, lr : 0.0001, scheduler decay 0.99, epoch 30, dropout 및 L2-regulization 0.1

3. GPU 전용메모리 사용량 및 예상 훈련시간
   
   RTX A6000 두 장으로 DataParallel을 이용할 경우 위의 셋팅으로는 8000MiB*2에 30초정도 걸리며, 

   같은 방식으로 유전자 18000개에 배치2로 할 경우(GPU하나당 배치1되게), 45500MiB*2에 940초가 걸림

   auto-mixed precision 까지 동원했을 때 현재 A6000으로는 유전자 20000개까지 가능

   (더 많은 유전자를 feature로 쓰려면 80GiB 짜리 GPU 혹은 Model Parallel 코딩이 필요함)      

---------------------------------------------------------------------------------------------------

### 환경을 복사해 주는 사람 쪽 작업

conda env export > environment4.yaml

### 환경을 받는 사람 쪽 작업

conda env create -n tc --file environment4.yaml (아직 아무 가상환경도 안 만들었을 때)

conda env update --file environment4.yml (이미 자기가 만든 가상환경이 있고 거기 들어가 있을 때)

--------------------------------------------------------------------------------------

## Transformer-(Multiclass)Classifier 모델을 훈련 및 검증하기 위한 데이터셋을 만든 과정

0. 모델에 필요한 X, Y 값만을 포함하는 pandas 데이터프레임 제작

   0-1. Xenabrowser에서 2개의 tsv 파일 다운로드

        TCGA-STAD.htseq_fpkm-uq_transposed.tsv 

        - 407 * 60484

        - 407명에 대해 각 유전자 ID마다의 mRNA발현량이 기록되어 있음

        - 60484개 중 일부를 X feature 로 삼음

        TCGA-STAD.GDC_phenotype.tsv

        - 544 * 115

        - 544명에 대해 115가지의 표현형 정보가 기록되어 있음

        - 이중에서 종양 악성등급에 대한 1개의 컬럼만을 Y label 로 삼음

   0-2. 파이썬 pandas 패키지를 이용하여 두 파일의 기증자 ID를 기준으로 Inner Join 하여 

        407 * 60486 크기의 통합된 데이터프레임을 만들고 중복되는 Ensembl_ID 컬럼을 삭제

   0-3. 1차로 위암에 관련된 유전자 1202개 리스트를 뽑고

        2차로 PHGDH 와 단백질 상호작용하는 유전자 369개 리스트를 뽑아 합치고,

        유전자ID를 모르는 17개를 빼서 총 1554개의 유전자 ID 리스트를 만듦

   0-4. 2번에서 만든 전체 데이터프레임의 컬럼이름이 3번 리스트에 존재하는 것만 추출하니

        여기서 다시 53개가 빠져 최종적으로 407 * 1501 크기의 데이터프레임이 나왔고

        이를 xy_STAD-related-only3.csv 와 xy_STAD-related-only3.pkl 로 저장함


1. 모델에 넣기 위한 최종 X, Y 값을 위한 numpy 데이터셋 제작 

   1-1. 유전자아이디에서 점이하를 버린 리스트를 컬럼으로 하는 데이터프레임을 만들고

        컬럼 이름 오름차순으로 정렬 후, 정답값이 되는 악성 종양 등급 별로 행 정렬

   1-2. X, Y 값에 해당하는 컬럼 추출 (407, 1501), (407,)

   1-3. 레이블 분포가 균일하게 되게 하면서 train : val : test = 7 : 2 : 1 로 나눔

        이 때 Y 값을 정수로 변환 G1->1, G2->2, G3->3, GX->0

   1-4. 트랜스포머 임베딩에 넣기 위해 X 값을 정수로 변환하는데  

        이 때 좀 더 풍부한 표현을 위해 min-max 정규화시키고 

        0 ~ 32 대신 0 ~ 320 사이의 정수로 변환함

   1-5. 데이터셋을 저장 (현재는 데이터가 모자라 train + test : val = 8 : 2 로만) 

        STAD_Dataset3_Train.h5, STAD_Dataset3_Test.h5

   1-6. 저장이 잘 되었는 지 확인

--------------------------------------------------------------------------------

### 모델 설계상의 아이디어
 
훈련데이터 feature인 유전자ID를 오름차순으로 정렬한다는 것을 문장에서 단어 배치를 맞게 하는 것에 비유하면,

현재 X_Train이 1501개의 데이터를 가진다는 것은 한 문장에 단어가 1501개 들어 있다는 것을 의미하며,

각 유전자 ID마다 0~X 사이의 정수를 가진다는 것은 단어가 표현할 수 있는 경우의 수가 X가지라는 것을 의미한다.

