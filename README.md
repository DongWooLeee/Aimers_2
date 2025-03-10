# LGAimers2

# 1. 프로젝트 개요

## 1.1 배경 및 목표
- **스마트 공장** 공정데이터를 활용하여 제품의 **품질 상태(Y_Class = 0,1,2)**를 예측하고자 함.  
- 데이터는 **결측치**가 매우 많고(라인별로 센서 이벤트가 달라 결측 발생), **클래스 불균형**도 존재.
- 
## 1.2 핵심 난제
1. **결측치 다량 존재**: 단순 평균/중위수로 채우기에 부담되는 스케일  
2. **라인별 센서 이벤트**가 달라, “비결측 열 집합”이 라인마다 상이  
3. **Y_Class**가 (0,1,2)로 불균형 + 0,2는 모두 ‘불량’으로 묶일 수 있는 구성 → 오분류 방지가 중요

---

# 2. 데이터 및 변수

## 2.1 제공 데이터
- **`train.csv`**  
  - 범주형: `PRODUCT_CODE`(A_31/T_31/O_31), `LINE`(6개 라인), `PRODUCT_ID`(고유 ID)  
  - 수치형: `X_1 ~ X_3326` (비식별 센서값)  
  - 타겟:
    - `Y_Quality`(연속형 지표)  
    - `Y_Class` (0/1/2)

- **`test.csv`**  
  - 동일 구조이나 `Y_Quality`, `Y_Class`가 제외됨(예측 대상)

## 2.2 라인별 특징
- 6개 라인이 서로 다른 조업조건, 센서환경 → 결측치 발생 양상 다름.  
- 특정 라인(예: 3,4)에서 뚜렷하게 **데이터 행 수가 적거나** 결측 패턴이 고르지 않은 경향 → 모델링 시 별도 전략 고려.

---

# 3. EDA (탐색적 데이터 분석)

## 3.1 범주형 변수

- **`PRODUCT_CODE`**  
  - A_31에서 불량률이 더 높고, T_31이 상대적으로 양호.  

- **`LINE`**  
  - 라인별 **Y_Class 분포**와 **Y_Quality 분포**가 달라, 단일 모델로 합쳤을 때 과적합/부적합 위험이 있을 수 있음.  

  > **초기에는** 라인별 Y_Class와 Y_Quality의 데이터 분포가 다르므로,  
  > 단일 모델을 적용하면 특정 라인의 특성이 **과도하게 반영**되거나 **무시될 가능성**이 있다고 판단.  
  > 따라서 **라인별 개별 모델을 학습하는 접근을 시도**했으나,  
  > **실험적으로 모든 라인을 통합한 모델로 학습을 진행하고, 3,4번 라인을 제외한 Validation Set에 대해 평균적으로 더 우수한 성능을 보였음.**  
  > 단, **라인 3·4**는 개별적으로 학습했을 때 성능이 개선되었으므로,  
  > 최종적으로 **(1·2·5·6)은 단일 Regressor를, (3·4)는 별도의 Classifier를 학습하는 혼합 모델 전략을 사용**.

---

## 3.2 라인 3·4의 데이터 특성 및 선택적 Feature 활용

> **EDA 결과**, 라인 3·4에서는 **불량품 비율이 상대적으로 매우 높아**,  
> Y_Class(0,1,2) 불균형이 다른 라인 대비 심각하게 나타남.  
> 예를 들어, **라인 3·4의 불량률이 평균적으로 30% 이상 높았으며**,  
> 단일 모델이 모든 라인을 학습할 경우 **불균형 데이터로 인해 라인 3·4의 불량 예측이 왜곡될 가능성이 존재**.  
>  
> **따라서**, 라인 3·4에 대해서는 **별도의 모델을 학습(fitting)하는 전략을 활용**하였으며,  
> 이 과정에서 **라인 3·4 데이터를 따로 Holdout하여 모델을 훈련**.  
>  
> - **데이터가 적을수록 Feature 개수를 줄이는 것이 필수적** → 과적합 방지  
> - **라인 3·4에서 관측된 Feature만 사용** → 모델이 의미 없는 변수를 학습하는 것을 방지  
> - **Feature Selection의 자연스러운 과정으로 볼 수 있음**  
>  
> 단, **라인 3·4의 데이터가 적기 때문에**, 과적합을 방지하기 위해  
> **모든 제품에서 공통으로 존재하는 Feature만 선택하여 학습**을 진행.  
> - 라인 3과 4에서 생산된 모든 제품들에서 **관측된 column만을 사용 (합집합 적용)**  
> - 기존 **3,000개의 Feature에서 약 80% 축소된 Feature로 학습**  
>  
> ✔ **Validation Set에서 성능 비교**  
> - 전체 데이터셋을 활용한 모델 → **Validation Score: 0.587**  
> - 라인 3·4를 별도로 Holdout한 모델 → **Validation Score: 0.784** (성능 향상)  
> - **5 Fold 평균값 기준, 여러 랜덤 시드에서도 일관된 향상된 결과를 보임**  
> - **Public Score 기준 39위 → 9위로의 가장 큰 성과를 달성**  

---

# 4. 전처리 (Preprocessing)

## 4.2 결측치 대체
- **모든 라인의 학습 모델에서는 결측치를 0으로 대체하여 처리.**  
- **평균(mean), 최빈값(mode) 대체도 실험했으나, 오히려 성능이 저하됨.**  

> **센서 데이터 특성상 결측값(NaN)은 실제 값이 존재하지 않음을 의미할 가능성이 높음.**  
> 따라서, **결측값을 0으로 채우는 것은 단순한 임퓨테이션이 아니라, 데이터를 모델이 더 쉽게 해석할 수 있도록 하는 전처리 과정으로 볼 수 있음.**  
>  
> 이후 추가적인 모델 경량화를 위해 **Feature Selection 기법(Feature Importance, L1 Regularization)을 고려**하였으나,  
> **현재는 0으로 임퓨테이션하는 것이 성능상 가장 우수하여 이를 Baseline으로 사용.**  

---

# 5. 모델링 (Modeling)

## 5.1 핵심 아이디어
- **CatBoost Regressor**를 통해 `Y_Quality`를 예측 후, **Threshold**를 이용해 `Y_Class`(0/1/2)로 변환  
  - (추가로, **Line3·4**는 별도의 Classifier 접근을 시도하여 더 나은 성능을 확보)

## 5.2 Threshold 설정 로직
## 5.2 Threshold 설정 논리

### 🔹 **Threshold 설정 배경**
> 본 프로젝트에서는 **Y_Quality(연속형 품질 지표)를 예측한 후, Threshold를 설정하여 Y_Class(0/1/2)로 변환**하는 방식으로 설계하였음.  
> Threshold 설정 시, 단순한 데이터 분포가 아닌 **공장의 품질 관리 목표**를 반영하여 **불량(0,2)이 정상(1)으로 잘못 분류되는 것을 막는 방향으로 조정**하였음.

📌 **공장에서의 목표:**  
1. **불량(0)과 품질 과다(2)를 최대한 줄이고, 정상(1) 제품을 정확하게 분류**  
2. **불량(0)이 정상(1)으로, 또는 품질 과다(2)가 정상(1)으로 잘못 분류되는 리스크를 줄이는 것이 중요**  
3. **따라서, 정상(1)의 범위를 좁히고 불량(0,2)을 보수적으로 분류하는 Threshold 설정**  

---

### 🔹 **Threshold 설정 방식**
```python
a = train_df[['Y_Class','Y_Quality']].groupby('Y_Class').agg(['mean','min','max','count'])

preds = []
for p in pre_preds:
  if p <= a[('Y_Quality','max')][0]:  
    preds.append(0)  # 품질 미달 (불량)
  elif p <= a[('Y_Quality','min')][2]:  
    preds.append(1)  # 정상 제품
  else:
    preds.append(2)  # 품질 과다 (불량)
```

### 📌 **Threshold 설정의 의미**
1. **품질 미달(0) 제품의 최대값을 기준으로 첫 번째 Threshold 설정**
   ```python
   if p <= a[('Y_Quality', 'max')][0]:
   ```
   ✅ **이유:**  
   - 품질 미달(0) 제품이 정상(1)으로 들어가지 않도록 보수적으로 설정  
   - 즉, **불량 제품이 정상으로 분류될 가능성을 최대한 차단**  

2. **정상(1) 제품의 최소값을 기준으로 두 번째 Threshold 설정**
   ```python
   elif p <= a[('Y_Quality', 'min')][2]:
   ```
   ✅ **이유:**  
   - 정상 제품(1)의 범위를 보수적으로 설정하여 **품질 과다(2) 제품이 정상(1)으로 들어가는 오류 방지**  

3. **나머지 모든 경우, 품질 과다(2)로 설정**
   ```python
   else:
       preds.append(2)  # 품질 과다 (불량)
   ```
   ✅ **이유:**  
   - 공정에서 **품질 과다(2)도 불량으로 간주되므로, 명확하게 구분하는 것이 중요**  
   - 만약 예측값이 1 Class의 최소값을 초과하면, **즉시 품질 과다(2)로 판단**  

---



---

# 6. 모델 검증 및 제출

## 6.1 Stratified K-Fold
- **불균형** 대응 차원에서 **Stratified** 분할로 Repeated K-Fold Validation 진행 (5 folds)
- SMOTE·ADASYN 등 오버샘플링은 큰 효과가 없었고, Overfitting 위험만 증가 → 미사용  

## 6.2 Submission Process
- Line 1,2,5,6 의 경우 전체 라인에 대해 학습을 진행한 모델을 사용하여 Regressor로 예측, Line 3,4는 Classifier로 진행
- 기존 `submission.csv` 파일을 활용하여 최종 제출  

---



# 7. To-Do + Additional Insights
- **Incremental Learning / 주기적 모델 업데이트**  
- **Threshold 조정 가능성 검토** (불량 최소화 목적)  
- **Feature Selection 최적화** → Tree 기반 모델을 사용하여 중요 변수를 선별, 모델 경량화 가능  
