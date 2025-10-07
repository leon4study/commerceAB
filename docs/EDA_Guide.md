# EDA(탐색적 데이터 분석) 순서 — 단계별 체크리스트와 실무 가이드

좋습니다. 데이터 분석에서 EDA는 모델링 전 **문제 정의·데이터 품질 점검·가설 생성**의 역할을 한다. 아래는 실무에서 자주 쓰이는 **단계별 순서**와 각 단계에서 구체적으로 해야 할 일, 추천 기법/코드 스니펫(파이썬·SQL), 주의점들을 정리한 것이다.

---

## 전체 순서(요약)

1. 목표/가설 정의
2. 데이터 수집·적재(ingest) 확인
3. 데이터 스냅샷(메타데이터) 확인
4. 데이터 품질 점검(결측·중복·타입·메모리)
5. 단변량 분석(수치·범주형 분포)
6. 이상치 탐지 및 처리
7. 이변량/상관관계 분석(타깃과의 관계 포함)
8. 다변량 패턴(차원축소·클러스터링 등)
9. 시계열·공간 데이터 특화 분석(해당 시)
10. 결측치 처리 전략 및 보간/대체(임퓨테이션)
11. 피처 엔지니어링(인코딩·변환·파생변수)
12. 최종 데이터셋 요약·보고서 작성(데이터 딕셔너리, 프로파일링)

아래에서 각 단계를 차근차근 풀어서 설명한다.

---

### 1) 목표(Problem)와 가설 정의

* **무엇을 예측/분석하려는가?** 예: "30일 내 이탈 예측" 또는 "주문 건수 변화의 주요 드라이버 식별"
* **평가지표는?** (AUC, RMSE, MAE, CTR 등)
* **가설을 미리 적어두기** — EDA는 가설 검증/갱신을 위한 과정이다.
* 산출물: 문제 정의 문서, KPI 목록, 샘플 가설(예: 특정 카테고리에서 재구매율이 높다)

---

### 2) 데이터 수집·적재 확인

* 데이터 출처(로그, DB, API, CSV)와 스키마를 확인.
* **샘플링 전략**: 전체가 크면 랜덤 샘플, 시간 기반 샘플(시간의존성 있을 때).
* **SQL 예제(레코드 요약)**:

```sql
SELECT COUNT(*) AS n_rows,
       COUNT(DISTINCT user_id) AS unique_users,
       MIN(event_time) AS min_time,
       MAX(event_time) AS max_time
FROM raw.events;
```

* 대용량일 때는 `LIMIT`, `SAMPLE`, 또는 chunked read 사용.

---

### 3) 데이터 스냅샷(메타데이터)

* 코드(파이썬):

```python
df.info()
df.head()
df.sample(5)
df.dtypes
df.memory_usage(deep=True).sort_values(ascending=False)
```

* 확인 항목: 컬럼명·타입·NA 비율·메모리 사용량·고유값 수(cardinality)

---

### 4) 데이터 품질 점검

* **결측치**: `df.isnull().mean()*100` — 컬럼별 결측률(%)
* **중복**: `df.duplicated().sum()`
* **타입 오류**: 수치로 되어야 할 컬럼에 문자열 등
* **카디널리티**: 범주형의 고카디널리티 판단(예: unique/n_rows > 0.5 → 주의)
* **메모리 최적화**: int/float downcast, category로 변환

```python
df['col'] = pd.to_numeric(df['col'], downcast='integer')
df['cat'] = df['cat'].astype('category')
```

* 산출 KPI: completeness(%), uniqueness, validity (포맷·범위 위반 건수)

---

### 5) 단변량 분석 (Univariate)

* **연속형**: 히스토그램, 박스플롯, ECDF → 중심(평균·중앙값), 분산, 왜도(skew), 첨도(kurtosis) 확인

  ```python
  df['x'].describe()
  df['x'].skew(), df['x'].kurt()
  ```
* **범주형**: 상위 카테고리 frequency, 비율, 희소 카테고리 병합(others)

  ```python
  df['cat'].value_counts(normalize=True).head(20)
  ```
* 시각화: 히스토그램, 막대그래프, 누적분포. **로그 스케일**이나 **박스-콕스(Yeo-Johnson)** 변환 고려.

---

### 6) 이상치(Outlier) 탐지 및 처리

* **IQR 규칙**: Q1-1.5*IQR, Q3+1.5*IQR
* **Z-score**: |z| > 3 (정규분포 가정)
* **MAD(robust)**: 중앙값 기반
* **모델기반**: IsolationForest, LocalOutlierFactor(LOF) — 고차원 데이터에 유용

```python
from sklearn.ensemble import IsolationForest
iso = IsolationForest(contamination=0.01).fit(df_num)
outliers = iso.predict(df_num)  # -1 이상치, 1 정상
```

* 처리 전략: 삭제, 윈저라이즈(극단값 컷), 변환(로그), 모델에서 robust 방법 사용.
* **주의**: 타깃이 희박한 케이스(예: fraud)에서는 '이상치'가 중요한 신호일 수 있으니 단순 제거 금지.

---

### 7) 이변량(Bivariate) 및 타깃 관계 분석

* **연속-연속**: Pearson(선형), Spearman(순위) → 산점도, 회귀선

  ```python
  df[['x','y']].corr(method='pearson')
  ```
* **범주-연속**: 그룹별 요약(평균·중앙값), 박스플롯, ANOVA 검정
* **범주-범주**: 교차표(crosstab), Chi-square, Cramér's V
* **이진 타깃과 연속**: point-biserial correlation
* **다중공선성**: VIF 계산 (VIF > 5~10 주의)

```python
from statsmodels.stats.outliers_influence import variance_inflation_factor
vif = [variance_inflation_factor(X.values, i) for i in range(X.shape[1])]
```

* 시각화: 상관행렬 히트맵, 산점도 매트릭스(작은 수 변수에 한함)

---

### 8) 다변량(Multivariate) 분석

* **차원 축소**: PCA(선형), t-SNE/UMAP(비선형) — 시각화·노이즈 제거·주성분 해석
* **클러스터링**: KMeans(구형클러스터), DBSCAN(밀도기반), 계층적 군집
* **상호작용 탐색**: feature interactions(교호작용) 생성 및 중요도 확인(모델 기반)
* 코드 예: PCA

```python
from sklearn.decomposition import PCA
pca = PCA(n_components=2).fit_transform(scaled_X)
```

---

### 9) 시계열·공간 데이터 특화 EDA (해당 시)

* **시계열**: 시계열 분해(Trend/Seasonality/Residual), ACF/PACF, 정상성 검사(ADF), 시차(lag) 분포 확인

```python
from statsmodels.tsa.seasonal import seasonal_decompose
seasonal_decompose(ts, period=7).plot()
```

* **공간**: 히트맵, 공간 클러스터링(Hotspot), 좌표 이상치

---

### 10) 결측치(Missing) 분석과 처리 전략

* **결측 유형**: MCAR, MAR, MNAR 판단 시도(도메인 지식 필요)
* **패턴 시각화**: `missingno` 라이브러리로 패턴 확인
* **임퓨테이션 방법**:

  * 단순: mean/median/mode
  * 고급: KNNImputer, IterativeImputer(MICE), 모델 기반 예측 임퓨테이션

```python
from sklearn.impute import KNNImputer, IterativeImputer
```

* **주의**: 훈련/테스트 분리 전에 임퓨테이션을 fit → leak 주의. 파이프라인 사용 권장.

---

### 11) 피처 엔지니어링(Feature Engineering)

* **인코딩**: One-hot(고유값 적음), Ordinal, Target/WOE encoding(범주형 수가 많을 때)
* **스케일링/변환**: StandardScaler, RobustScaler, PowerTransformer
* **파생변수**: 날짜→요일/시간대/주말/휴일, 로그/제곱근, 비율·증감률 등
* **범주 합치기**: 희소 카테고리 → 'OTHER'로 병합
* **피처 선택**: 상관·VIF·모델 중요도(LASSO, tree-based) 활용

---

### 12) 최종 데이터셋 요약 및 보고서

* **데이터 딕셔너리**: 컬럼 설명, 타입, 범위, 결측률, 대표값 예시
* **요약 보고서**: 주요 통찰(Top 5), 처리한 결측/이상치 내역, 추천 피처 리스트, 모델링 시 고려사항(데이터 분포·스케일)
* **자동화 도구**: ydata-profiling(pandas-profiling), Great Expectations, D-Tale — 대시보드/HTML 리포트 생성
* 산출물: `EDA_report.html`, `data_dictionary.csv`, 재현 가능한 노트북(.ipynb)

---

## 실무 팁(핵심)

* **샘플링**: 전체 데이터가 크면 먼저 대표 샘플로 EDA → 필요시 청크 기반 처리.
* **재현성**: 모든 전처리(임퓨테이션, 스케일링)는 파이프라인에 넣고 저장(파라미터 포함).
* **타임라인(시간의존성)**: 모델링 시 미래 정보를 사용하지 않도록 시계열 순서 유지.
* **데이터 누수(Leakage)**: 타깃에 영향 주는 파생변수는 모델링 단계에서 엄격히 검토.
* **문서화**: EDA에서 얻은 인사이트는 반드시 코드/리포트로 남기기(면접·협업에 유용).

---

## 예시: 빠른 파이썬 체크(핵심 코드 스니펫)

```python
# 결측률, 유일값, 타입 확인
meta = pd.DataFrame({
  'dtype': df.dtypes.astype(str),
  'n_missing': df.isnull().sum(),
  'pct_missing': df.isnull().mean()*100,
  'n_unique': df.nunique()
}).sort_values('pct_missing', ascending=False)

# 연속형 분포 요약
num_cols = df.select_dtypes(['number']).columns
df[num_cols].describe().T

# 상관행렬
corr = df[num_cols].corr(method='spearman')  # 비선형 관계도 확인
```

---

## 흔한 실수와 회피법

* **무작정 이상치 제거**: 이상치가 비즈니스적 신호일 수 있음 → 도메인 검증 후 처리
* **단순 평균 임퓨테이션만 사용**: 분포 왜곡 가능 → 분포 보존을 고려한 방법 추천(MICE, KNN)
* **제대로 된 샘플링 없이 전체 통계 사용**: 대규모 데이터는 메모리·시간 문제 → 샘플링·분산처리(Dask, Vaex) 사용
* **데이터 딕셔너리 미작성**: 협업·재현성 저해

