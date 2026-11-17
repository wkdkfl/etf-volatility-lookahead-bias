# ETF Volatility Forecasting and Look-Ahead Bias

**뉴스 텍스트와 금융지표를 결합한 ETF 변동성 예측과 미래정보 편향: 미국과 영국 시장의 비교 연구**
*ETF Volatility Forecasting and Look-Ahead Bias with News Text and Financial Indicators: A Comparative Study of the US and UK Markets*

석사학위논문의 실험 코드와 데이터입니다. 뉴스 헤드라인(FinBERT)과 금융지표로 ETF 실현변동성을 예측하되, **자료 구축 단계의 편향이 성능 수치를 얼마나 부풀리는지**를 같은 파이프라인 안에서 수치로 측정하고, 편향을 통제한 뒤 실제로 남는 예측력을 검증합니다.

---

## 연구 동기

초기 파이프라인에서 5일 실현변동성 R²가 **영국 0.93 / 미국 0.83**, 5일 방향성 AUC가 **0.84 / 0.75**로 나왔습니다. 일별 금융 시계열로는 비현실적인 수치입니다.

원인을 추적해 보니 원자료가 **달력일 기준**이었습니다. 주말·공휴일 종가가 직전 거래일 값으로 채워져 비거래일 수익률이 정확히 0이 되었고, 모형은 시장 동학이 아니라 *"내일이 주말인가"* 를 학습하고 있었습니다. 거래일 필터와 시점 정렬을 적용하자 미국은 R² 0.363, AUC 0.587로 떨어졌고 영국은 유의성이 사라졌습니다.

→ 모형은 그대로 두고 **자료 구축만 바꿨는데 결론이 달라졌습니다.** 이 왜곡의 크기와, 왜곡을 걷어낸 뒤 남는 예측력이 이 연구의 질문입니다.

## 연구질문과 결과 요약

| 연구질문 | 결과 |
|---|---|
| **RQ1** 편향 통제 후 변동성을 유의하게 예측할 수 있는가? | 미국: 5일 RV R² **0.363** (95% CI 0.140–0.476), HAR-RV 0.187 · GJR-GARCH 0.152보다 높음. 영국: 모든 피처군의 CI가 0을 포함함. 시험 구간이 변동폭이 가장 좁은 국면이었던 영향이 큼 |
| **RQ2** 자료 구축 편향은 성능 평가를 얼마나 왜곡하는가? | 편향의 크기(ΔR² 0.47–0.67)가 모형 개선폭(HAR-RV 대비 ΔR² 0.176)의 **2.7–3.8배**. 방향성 AUC는 통제 후 0.543–0.556 수준만 남음 |
| **RQ3** 뉴스는 가격·금융지표 대비 추가 예측력을 주는가? | 금융지표 단독·뉴스 단독보다는 결합 모형이 유의하게 우월. 가격 자기회귀(AR) 대비 증분은 유의하지 않음(뉴스 커버리지를 보강해 재검정해도 같음) |
| **RQ4** 두 시장에서 예측 가능성은 어떻게 나타나는가? | pooled 학습 시 미국 R² 0.222 → 0.363. 표본 증가 효과와 분리되지 않아 보조적 관찰로 한정 |

> 편향을 통제하면 방향성 신호는 거의 사라지고, 변동성 예측력은 제한적이지만 유의하게 남는다.

## 방법 개요

- **대상**: SPY (S&P 500), ISF.L (FTSE 100) · 2014-01 ~ 2025-12
- **뉴스**: 금융 헤드라인 미국 22,971건 / 영국 36,739건. 경제 키워드 사전(6,670개)으로 필터링하고 FinBERT로 일별 감성 통계와 임베딩(PCA 15차원)을 만듦
- **피처**: 71개 (자기회귀 25 · 금융지표 15 · FinBERT 감성 16 · 임베딩 PCA 15). 피처군 5종 `Financial_Only` · `News_Pure` · `AR_Only` · `AR+News` · `Full`
- **편향 통제**
  - 거래일 필터: 달력일 전방보간 행 제거 (`_trading_days.py`)
  - 비대칭 시차: 금융지표는 t-1 종가까지만, 뉴스는 장 마감 이전 기사만 사용
  - 시간대 정렬: NYSE는 LSE보다 약 4.5시간 늦게 마감하므로, 영국 모형의 미국 기반 변수는 1일 시차를 두고 같은 시간대의 EU 지수 5종으로 보완
- **평가**: 시간순 단일 분할(70/15/15) → 8-fold 확장 Walk-Forward → 중첩 Walk-Forward. 고정 하이퍼파라미터 LightGBM과 pooled 학습, 검증 구간 Mincer-Zarnowitz 보정 사용. 블록 부트스트랩 95% CI(블록 10, 5,000회)와 Diebold-Mariano 검정(HLN 보정)으로 판정
- **벤치마크**: HAR-RV · GARCH(1,1) · GJR-GARCH를 같은 표본·분할·척도에서 다시 추정

## 저장소 구조

```
├── 공통데이터/                       # 원천 데이터와 수집·구축 코드
│   ├── collect_economic_news.py     # 경제 뉴스 수집
│   ├── USD/  UK/                    # 시장별 가격·환율·VIX·금·원유·국채 CSV, 뉴스 CSV
│   │   ├── *_headline_crawling_code.ipynb   # 헤드라인 크롤링 (API 키는 제거됨)
│   │   └── *_databuilder.ipynb              # 연구용 데이터셋 구축
│   └── USD/download_finbert.py      # FinBERT 모델 다운로드
│
├── 01_실패실험_수익률예측/            # 초기 실험: 수익률 예측 (편향 통제 이전)
│   └── USD/ UK/  *_modeling.ipynb, results/
│
└── 02_성공실험_변동성방향성예측/       # 본 실험: 변동성·방향성 예측 + 편향 통제
    ├── _trading_days.py             # 거래일 필터 (핵심 편향 통제)
    ├── _build_*_features*.py        # FinBERT 뉴스 피처 생성
    ├── _final_v10.py                # ★ 최종 변동성 실험 (피처군별 R², DM 검정)
    ├── _final_v10_direction.py      # ★ 최종 방향성 실험 (AUC, DM 검정)
    ├── _save_folds_v10.py           # 8-fold Walk-Forward
    ├── _garch_aligned_v6.py         # HAR-RV / GARCH / GJR-GARCH 벤치마크
    ├── _pooled_calibrated_v9.py     # pooled vs 단독 학습 (RQ4)
    ├── _robustness_v13.py           # 강건성: FTSE 250 대체 표적, fold 성능 설명
    ├── _direction_v16~v18_*.py      # 방향성 Walk-Forward / 중첩 Walk-Forward / F1
    ├── _run_reexperiment.py         # 재실험 러너 (뉴스 데이터·PCA 솔버 변형별 실행)
    ├── _explore_targets_v5.py, _regime_fixes_v7.py, _improve_v11.py, ...  # 중간 탐색 실험
    ├── USD/ UK/                     # 연구용 데이터셋, 뉴스 임베딩(npy), results_* 결과
    ├── 재실험_결과/, 방향성개선_v15~v18/, 개선실험/   # 실험 결과 CSV
    └── requirements_reexperiment.txt
```

파일 이름의 `v4`~`v18`은 실험 버전입니다. 논문의 최종 수치는 **v10 프로토콜**(`_final_v10*.py`, `_save_folds_v10.py`, `_garch_aligned_v6.py`, `_pooled_calibrated_v9.py`)과 방향성 중첩 평가(`_direction_v17_*.py`)에서 나왔습니다.

## 실행 방법

```bash
cd 02_성공실험_변동성방향성예측
pip install -r requirements_reexperiment.txt

python _final_v10.py              # 변동성: 피처군별 R² + DM 검정
python _final_v10_direction.py    # 방향성: AUC + DM 검정
python _save_folds_v10.py         # 8-fold Walk-Forward
python _garch_aligned_v6.py       # 전통 모형 벤치마크
python _pooled_calibrated_v9.py   # pooled vs 단독 학습
```

환경 변수로 실험 변형을 바꿀 수 있습니다 (`_news_variant.py` 참고).

```bash
NEWS_DATA=new PCA_SOLVER=full RESULT_DIR=<출력폴더> python _final_v10.py
```

### 참고 사항

- 원 실험은 Python 3.7.4 / scikit-learn 1.0.2에서 수행했습니다. `requirements_reexperiment.txt`는 재실험 환경(Python 3.12)입니다. PCA 솔버 차이 때문에 뉴스가 포함된 피처군의 수치가 소수점 단위로 다를 수 있습니다.
- 논문 그림 생성 스크립트는 이 저장소에 포함하지 않았습니다. `_run_reexperiment.py`를 전체 단계로 실행하면 그림 단계(`_figs_8_10_v10.py`)에서 멈추므로 `--steps`로 필요한 단계만 지정하세요.
- 일부 스크립트 주석이 가리키는 `plan/*.md`(사전 고정 프로토콜 문서)는 포함하지 않았습니다.
- 크롤링 노트북을 다시 실행하려면 각 뉴스 API 키를 직접 발급받아 `YOUR_API_KEY` 자리에 넣어야 합니다.
- 뉴스 헤드라인 데이터는 연구 목적으로 수집했습니다. 각 원 제공처의 이용 약관을 따릅니다.
