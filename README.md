# BRFSS 건강 지표 기반 당뇨 라벨 분류

미국 CDC의 2015년 BRFSS 건강 설문에서 이미 기록된 라벨을 BMI, 연령 구간, 일반 건강 상태 등의 지표로 분류하는 실험입니다. [Jupyter 노트북](prediction_of_diabetes_patient.ipynb)에서 XGBoost 학습부터 독립된 테스트 세트 평가까지 실행할 수 있습니다.

## 데이터

[Kaggle Diabetes Health Indicators Dataset](https://www.kaggle.com/datasets/alexteboul/diabetes-health-indicators-dataset)에서 `diabetes_binary_5050split_health_indicators_BRFSS2015.csv`를 내려받아 노트북과 같은 폴더에 둡니다. 이 파일은 [CDC의 2015 BRFSS 설문](https://www.cdc.gov/brfss/annual_data/annual_2015.html)을 바탕으로 두 클래스의 비율을 50:50으로 맞춘 가공 데이터입니다. CSV는 저장소에 포함되지 않습니다.

예측 대상은 `Diabetes_binary`이며, 정수 0과 1로 변환해 사용합니다. 입력에서는 라벨을 제외하고, 이전 실험과 같은 입력 열을 유지하기 위해 `AnyHealthcare`도 제외합니다.

## 실행 방법

Python 3.13 가상 환경을 만들고 활성화한 뒤 저장소 루트에서 실행합니다.

```sh
python -m pip install -r requirements.txt "jupyterlab>=4,<5"
python -m jupyterlab
```

[노트북](prediction_of_diabetes_patient.ipynb)을 열고 첫 셀부터 순서대로 실행합니다. Colab에서 열 때는 CSV를 현재 작업 폴더에 직접 업로드하세요. macOS에서 XGBoost가 `libomp.dylib`를 찾지 못한다면 [Homebrew의 libomp](https://formulae.brew.sh/formula/libomp)를 설치하세요.

## 실험 흐름

1. **분할** — 라벨 비율을 유지하며 원본을 학습 60%, 검증 20%, 테스트 20%로 나눕니다. BMI 이상치 제거, 표준화, SMOTE를 적용하지 않습니다.
2. **모델 선택** — 학습 세트에서만 층화 5겹 교차 검증을 수행합니다. 각 XGBoost 모델을 300회 부스팅하고, 8개 파라미터 조합 중 평균 ROC AUC가 가장 높은 조합을 선택합니다.
3. **검증** — 선택한 조합으로 다시 학습할 때는 부스팅 상한을 1,000회로 늘립니다. 검증 세트의 ROC AUC가 25회 연속 개선되지 않으면 학습을 멈추고, 검증 ROC AUC가 가장 높았던 시점의 모델을 사용합니다. 최적 시점이 1,000회 상한과 같으면 상한을 늘려 확인하라는 메시지가 표시됩니다. 같은 검증 세트에서 후보 임계값 0.25~0.65 중 F1이 가장 높은 값을 고릅니다. 동점이면 정밀도가 높은 값, 그래도 같으면 더 낮은 임계값을 선택합니다.
4. **최종 평가** — 모델과 임계값을 고정한 뒤 테스트 세트에서 정확도, 정밀도, 재현율, F1, ROC AUC와 혼동 행렬을 확인합니다.

그래프는 `outputs/`에 저장됩니다. `boxplot.png`는 학습 세트의 BMI 분포, `Recall_precision_value.png`는 검증 세트의 임계값별 정밀도·재현율, `ROC_AUC_curve.png`는 테스트 세트의 ROC 곡선과 혼동 행렬을 담습니다.

## 결과 해석

원본 CSV 70,692행과 노트북의 기본 시드(`156`)로 실행한 결과, 검증 세트에서 **부스팅 347회**와 **분류 임계값 0.35**를 선택했습니다. 이를 고정한 뒤 테스트 세트 14,139행에서 다음 점수를 얻었습니다.

| 정확도 | 정밀도 | 재현율 | F1 | ROC AUC |
| --- | --- | --- | --- | --- |
| 0.7410 | 0.6817 | 0.9042 | 0.7773 | 0.8327 |

평가는 2015년 설문을 바탕으로 구성한 50:50 표본 안에서 이루어집니다. 이 표본의 클래스 비율은 실제 인구의 유병률과 다르므로, 정밀도나 모델이 출력한 확률을 실제 인구에 그대로 적용할 수 없습니다. 이 실험은 미래 발병이나 개인 진단을 검증하지 않습니다.
