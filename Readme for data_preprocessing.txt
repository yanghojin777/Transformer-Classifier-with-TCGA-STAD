Readme for data_preprocessing 

Transformer-(Multiclass)Classifier 모델을 훈련 및 검증하기 위한 데이터셋을 만든 과정

1. Xenabrowser에서 2개의 tsv 파일 다운로드
   TCGA-STAD.htseq_fpkm-uq_transposed.tsv 
   - 407 * 60484
   - 407명에 대해 각 유전자 위치마다의 mRNA발현량이 기록되어 있음
   - 60484 개의 X feature 로 삼을 예정
   TCGA-STAD.GDC_phenotype.tsv
   - 544 * 115
   - 544명에 대해 115가지의 표현형 정보가 기록되어 있음
   - 이중에서 종양 악성등급에 대한 1개의 컬럼만을 Y label 로 삼을 예정

2. 파이썬 pandas 패키지를 이용하여 두 파일의 기증자 ID를 기준으로 Inner Join 하여 
   407 * 60486 크기의 통합된 데이터프레임을 만듦

3. Y label 데이터인 neoplasm_histologic_grade 가 'GX' 인 9행을 제거하고
   중복되는 키 값인 Ensembl_ID 컬럼을 삭제하여
   398 * 60485 크기의 데이터를 xy_full.csv 와 xy_full.pkl 파일로 저장함 

4. 추가적으로, X feature 개수가 너무 많아 문제가 될 것이 우려되어,
   오직 위암에 관련된 유전자위치만을 조사선정 후 관련 위치만을 담은 
   398 * 34 크기의 데이터를 xy_STAD-related-only.csv 와 xy_STAD-related-only.pkl 로 저장함

