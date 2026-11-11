# BRFSS 건강 지표 기반 당뇨 분류

미국 CDC의 2015년 BRFSS 건강 설문 데이터를 가공한 CSV를 사용해, 건강 지표로부터 데이터셋의 당뇨 여부 라벨을 분류하는 머신러닝 실험입니다. [Jupyter 노트북](prediction_of_diabetes_patient.ipynb)에 데이터 전처리, XGBoost 학습, 분류 임계값 비교를 담았습니다.

## 데이터

[Kaggle Diabetes Health Indicators Dataset](https://www.kaggle.com/datasets/alexteboul/diabetes-health-indicators-dataset)에서 `diabetes_binary_5050split_health_indicators_BRFSS2015.csv`를 내려받아 노트북과 같은 폴더에 둡니다. CSV는 저장소에 포함되어 있지 않습니다. 이 파일은 [CDC의 2015 BRFSS 설문](https://www.cdc.gov/brfss/annual_data/annual_2015.html)을 바탕으로 가공했으며, 두 클래스의 비율을 50:50으로 맞춘 버전입니다.

노트북은 `Diabetes_binary`를 예측 대상으로 사용합니다. BMI, 연령 구간, 일반 건강 상태 등의 지표를 입력에 사용하며, 라벨과 `AnyHealthcare` 열은 입력에서 제외합니다.

## 실행 방법

1. Python 3.11 가상 환경을 만들고 활성화합니다.
2. 저장소 루트에서 아래 명령을 실행합니다.

   ```sh
   python -m pip install jupyterlab "numpy<2" "pandas<3" "scikit-learn==1.3.2" "imbalanced-learn==0.11.0" "xgboost==1.7.6" matplotlib seaborn
   python -m jupyterlab
   ```

3. [prediction_of_diabetes_patient.ipynb](prediction_of_diabetes_patient.ipynb)를 열고 첫 코드 셀부터 실행합니다. 5겹 교차 검증으로 여러 조합을 학습하므로 파라미터 탐색에 시간이 걸릴 수 있습니다.

노트북은 `XGBClassifier.fit()`에 `eval_metric`과 `early_stopping_rounds`를 전달하는 기존 API를 사용합니다. 위 설치 예시는 이 호출 형식과 호환되는 버전을 지정합니다. [XGBoost 1.7.6 API 문서](https://xgboost.readthedocs.io/en/release_1.7.0/python/python_api.html#xgboost.XGBClassifier.fit)

## 실험 흐름

1. **전처리** — BMI 이상치를 1.5×IQR 기준으로 제거하고, BMI·연령 구간·소득 등 7개 열을 `StandardScaler`로 표준화합니다.
2. **데이터 분할** — 학습 80%, 테스트 20%로 나눈 뒤 학습 데이터에만 SMOTE를 적용합니다.
3. **학습** — 재현율을 기준으로 `GridSearchCV`에서 XGBoost 파라미터를 탐색합니다. 이후 별도로 지정한 파라미터로 모델을 다시 학습합니다.
4. **평가** — 분류 임계값 0.25~0.65에서 혼동 행렬, 정확도, 재현율, F1을 비교하고 예측 확률로 ROC AUC를 계산합니다. BMI 등 주요 변수의 상자 그림(`boxplot.png`), ROC 곡선(`ROC_AUC_curve.png`), 임계값별 정밀도·재현율 그래프(`Recall_precision_value.png`)를 저장합니다.

## 결과 해석

현재 노트북은 데이터를 분할하기 전에 이상치 기준과 표준화 통계를 계산합니다. 또한 교차 검증 전에 SMOTE를 적용하고, 조기 종료에도 테스트 세트를 사용합니다. 이 때문에 출력 지표가 독립적인 데이터에서의 성능보다 높게 나올 수 있습니다. 50:50으로 구성한 표본은 실제 유병률을 반영하지 않으므로, 이 실험의 출력은 학습용이며 개인의 진단에 사용할 수 없습니다.
