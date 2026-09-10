---
name: python-ml-pipeline-conventions
description: Python 기반 ML 모델·파이프라인의 코드, YAML 설정, 산출물을 추가·수정할 때 폴더 책임, 직접 YAML 설정, 단순한 설계, 주석 스타일을 일관되게 적용한다. Hydra나 실험 프레임워크를 도입하지 않는 경량 레포에 사용한다.
---

# Python ML Pipeline Conventions

## 목적과 범위

- 현재와 이후의 모델·알고리즘 파이프라인을 독립적으로 추가·수정할 수 있는 공통 구조를 제공한다.
- 기준 구조는 `root 진입점 → src 역할별 모듈 → configs → 산출물`이다.
- 이 레포는 제품·운영 코드다. **Hydra, 실험 프레임워크, CLI 파서, 필요 전의 범용 추상화는 도입하지 않는다.**
- 현재 코드의 모델·알고리즘 구현은 참고 사례이지, 새 모델에 강제할 설계가 아니다.

## 표준 구현 우선

- 대부분의 코드는 Python·PyTorch·pandas의 표준적이고 직선적인 구현을 따른다.
- 사용자가 요청하거나 베이스라인이 필요성을 보이지 않는 한, 고도화된 커스터마이징을 추가하지 않는다.
- 새 class hierarchy, factory, adapter, dependency injection, plugin registry, event system, logging framework, 범용 validation framework를 선제적으로 만들지 않는다.
- 예외 처리는 외부 경계에만 둔다.
  - 설정 파일·입력 파일·checkpoint 존재 여부
  - 사용자 설정값의 유효 범위
  - 외부 schema·열 순서·shape 계약
- 내부 코드에서 일어나지 않아야 할 상태를 넓은 `try/except`, silent fallback, 자동 보정으로 숨기지 않는다. 원인이 드러나는 기본 예외를 우선한다.
- 구 checkpoint·외부 형식 호환처럼 실제 운영 계약이 있을 때만 명시적인 호환 처리를 추가한다.
- 단순한 표준 코드와 추가 설계 중 선택이 애매하면, 코드 추가 대신 사용자에게 필요성을 확인한다.

```python
# 기본: 입력 계약이 깨지면 원인을 그대로 알린다.
if not model_path.exists():
    raise FileNotFoundError(f'가중치 없음: {model_path}')

# 금지: 요청 없는 fallback과 오류 은폐다.
try:
    model.load_state_dict(torch.load(model_path))
except Exception:
    model = make_default_model()
```

## 템플릿화 전 베이스라인 게이트

기존 모델·알고리즘을 템플릿화, 리팩터링, 다른 모델군으로 확장하기 전에 아래 순서를 반드시 지킨다.

1. **권위 있는 베이스라인을 확정한다.**
   - 사용자가 기준으로 삼는 것이 notebook인지, 특정 `.py` 실행 파일인지, 저장된 checkpoint인지 묻는다.
   - 파일이 여러 개이거나 notebook과 `.py`의 내용이 다르면 추측으로 하나를 고르지 않는다.
   - 기준 파일, config, 데이터 버전, checkpoint, 결과 디렉터리를 함께 기록한다.

2. **알고리즘의 세부 의도를 확인한다.**
   - 목적함수, 입력·출력 schema, 데이터 split, 정규화, seed, 초기화, 학습/탐색 횟수, early stopping, 양자화·후처리, 평가 지표와 평가 구간을 확인한다.
   - 제어 파라미터·도메인 파라미터·KPI처럼 의미가 바뀌면 결과가 달라지는 값은 이름만 보고 일반화하지 않는다.
   - 문서·코드·사용자 설명이 충돌하거나 핵심 값이 빠지면 템플릿화를 멈추고 사용자에게 구체적으로 다시 묻는다.

3. **베이스라인 재현 기준을 정한다.**
   - 비교 대상은 최소한 최종 metric, 평가 데이터/구간, seed, 산출물 schema다.
   - 확률·GPU 연산처럼 완전한 bitwise 일치가 불가능하면 허용 오차와 비교 방법을 사용자와 정한다.
   - 예: `test RMSE ≤ baseline + 0.01`, KPI 3종 소수 첫째 자리 일치, 추천 제어 파라미터가 bounds·JSON schema를 만족.

4. **변경 전후를 같은 조건으로 검증한다.**
   - 같은 config, 입력 데이터, checkpoint/초기화, seed로 baseline과 템플릿화 결과를 실행한다.
   - metric·예측값·추천값·생성 파일을 비교하고, 차이가 있으면 원인을 확인하기 전까지 “동일하다”고 주장하지 않는다.
   - 실행 환경이나 데이터가 없어 검증할 수 없으면, 검증하지 못한 조건과 필요한 실행 명령을 명시한다.

5. **검증 후에만 구조를 일반화한다.**
   - 기능 보존 리팩터링과 알고리즘 변경을 한 변경에 섞지 않는다.
   - 새로운 옵션·추상화는 baseline 결과가 보존된 뒤 별도 변경으로 추가한다.

### 사용자에게 확인할 질문 형식

베이스라인이 불명확할 때는 다음처럼 한 번에 핵심만 묻는다.

> 템플릿화 기준은 어느 파일인가요? (`notebooks/...ipynb` 또는 실행 `.py`) 그리고 동일 성능 판정에 사용할 데이터·config·seed·핵심 metric과 허용 오차를 알려주세요.

- 사용자가 기준을 지정하면 그 파일과 직접 호출하는 코드부터 읽는다.
- 사용자가 “현재 결과 그대로”라고만 말하면, 확인 가능한 최근 산출물과 config를 제시해 판정 기준을 확정한다.

## 먼저 확인할 것

- 수정할 폴더의 `README.md`, 가까운 실행 파일, 관련 YAML을 읽는다.
- 유사 구현과 모든 호출부를 `rg`로 찾는다.
- public 함수·YAML 키·CSV/JSON 열 이름·저장 경로는 외부 계약이다.
- 새 모델이 기존 모델과 입력·출력·평가 계약을 공유하는지 판단한다. 공유하지 않으면 기존 내부에 억지로 끼워 넣지 않는다.

## 폴더 역할

| 위치 | 역할 | 예시 |
|---|---|---|
| 루트 `*.py` | 실행 조립, YAML 읽기, 모델·트레이너 호출 | `train_forecast.py` |
| `configs/<pipeline>/` | 모델·학습·입력·출력·탐색 설정 | `configs/forecast/train.yaml` |
| `src/data/` | 입력 로드, 전처리, 도메인 지표, 입력 재현 | `forecast_preprocessing.py` |
| `src/nn/` | 재사용 가능한 순수 네트워크 블록 | `MLP.py` |
| `src/models/` | 모델별 `nn.Module`, forward, 모델 고유 update/evaluate | `MLP_prediction_model.py` |
| `src/trainer/` | epoch/batch 학습 루프, 체크포인트, 학습 평가 | `forecast_trainer.py` |
| `src/optimizer/` | 학습된 surrogate 또는 목적함수 위의 탐색 | `forecast_random_search.py` |
| `src/utils/` | 둘 이상 상위 모듈에서 쓰는 작은 독립 보조 기능 | `fix_seed.py` |
| `src/visualization/` | 결과를 받아 그림 하나를 만드는 leaf 함수 | `plot_forecast_curve.py` |
| `results/`, `figs/`, `saved_models/` | 실행 산출물 | `curve.csv`, `.pt` |
| `notebooks/` | 탐색·검증·분석 | 재현식 검증 notebook |

### 새 모델·파이프라인 추가

- 데이터 형식과 학습 흐름이 기존과 같으면 새 model 파일과 기존 trainer/config 패턴을 재사용한다.
- 계약이 다르면 필요한 경계만 독립시킨다.

```text
configs/<new_pipeline>/
root entrypoint(s)
src/data/<new_domain>.py       # 필요할 때만
src/models/<new_model>.py
src/trainer/<new_trainer>.py   # 기존 trainer와 계약이 다를 때만
src/visualization/<new_plot>.py
```

- 기존 파이프라인 파일을 일반 모델의 base class로 바꾸지 않는다.
- 실제로 둘 이상의 모델이 같은 계약을 공유할 때만 공통 클래스·factory·registry를 만든다.
- 새 파이프라인의 경계·입력·출력·산출물은 해당 폴더 README와 루트 README에 추가한다.

### 의존성 방향

```text
root entrypoint → data / models / trainer / optimizer / visualization
models           → nn / data / utils
trainer          → models / data / utils
optimizer        → models / data / utils
visualization    → data / utils
```

- 순환 import를 만들지 않는다.
- 예외가 불가피하면 구조상 이유를 해당 폴더 README에 남긴다.

## 설정과 진입점

- YAML이 실행 조건의 단일 출처다. 파라미터를 코드 상수·CLI·YAML에 중복 정의하지 않는다.
- Hydra를 사용하지 않는다. `yaml.safe_load()`로 직접 읽는다.
- 기본 YAML은 실행 파일에 명시하고, 다른 설정은 환경변수로 파일 경로만 교체한다.
- 설정 해석·형변환은 `if __name__ == '__main__':` 안에서 수행하고, `main()`에는 해석된 값을 넘긴다.

```python
if __name__ == '__main__':
    cfg_path = os.environ.get('PIPELINE_CONFIG', DEFAULT_CONFIG)
    with open(cfg_path, 'r') as f:
        cfg = yaml.safe_load(f)

    model_cfg = cfg['model']
    path_cfg = cfg['path']
    epochs = int(cfg['train']['epochs'])
    main(model_cfg=model_cfg, path_cfg=path_cfg, epochs=epochs)
```

- 진입점은 순서를 보이게 유지한다: 설정 읽기 → 입력 검증/로딩 → 모델 생성/로드 → 실행 → 산출물 저장.
- 실행 YAML은 결과 디렉터리에 함께 저장한다.

## 루트 실행 파일 템플릿

루트의 `train_*.py`, `inference_*.py`, `optimize_*.py`, `preprocess_*.py`는 파이프라인을 **조립**하는 파일이다. `src/`의 계산을 복사하지 않는다.

### 공통 규약

- 파일명은 동사와 대상이 드러나게 짓는다. 예: `train_<pipeline>.py`, `inference_<pipeline>.py`, `optimize_<algorithm>.py`.
- 모듈 docstring은 목적·기본 YAML·주요 산출물만 짧게 적는다.
- `main()`은 실행 한 번의 순서를 읽을 수 있게 유지한다. 설정 dict 해석은 `__main__`에 두고, `main()`에는 형변환·검증된 값을 넘긴다.
- root 파일에서 허용하는 일: 입력 존재·shape·열 순서 검증, 경로 조립, 모델/트레이너/optimizer 생성, 함수 호출, 산출물 저장·요약 출력.
- root 파일에서 금지하는 일: 모델 layer·loss 수식·학습 epoch loop·KPI 수식·feature engineering·후보 탐색 loop의 직접 구현.
- `Hydra`, `argparse`, WandB 등 별도 실행 프레임워크를 도입하지 않는다. YAML 직접 로딩과 환경변수 기반 설정 파일 교체를 유지한다.
- 원인을 숨기는 broad `except`나 `print` 후 조기 `return` 대신, 입력·설정 오류는 경로와 키를 포함한 구체적 예외로 실패시킨다.

### 학습 진입점: `train_<pipeline>.py`

순서: 설정 → 출력 위치/설정 snapshot → seed·device → 데이터 로드·검증 → 데이터 준비 → 모델·trainer 생성 → `trainer.train()` → 사후 지표·요약.

```python
def main(train_data, validation_data, test_data, *, model_cfg: dict,
         trainer_cfg: dict, paths: dict, seed: int, device: str) -> None:
    fix_seed(seed)
    model = build_model(model_cfg, device=device)
    trainer = ExampleTrainer(**trainer_cfg, out_dir=paths['out_dir'])
    result = trainer.train(model, train_data, validation_data, test_data)
    write_summary(result, paths['out_dir'])
```

- 데이터 준비가 재사용되거나 복잡하면 `src/data/` 함수로 옮긴다. root에는 단계 호출만 남긴다.
- 모델 생성에 필요한 config 값은 명시적으로 전달한다. `main()` 또는 trainer가 전역 `cfg`를 다시 읽지 않는다.
- trainer가 checkpoint·curve를 저장한다면 실행 파일은 같은 파일을 중복 저장하지 않는다.
- 실행 전 또는 직후에 사용한 YAML을 결과 디렉터리의 `config.yaml`으로 남긴다.

### 추론 진입점: `inference_<pipeline>.py`

순서: 설정 → 입력 검증 → 모델 metadata/weight 로드 → 입력을 학습 계약에 맞게 변환 → 예측 → 도메인 지표·그림 → 결과 저장.

```python
def main(model_path: str, raw_input, *, model_cfg: dict,
         output_dir: str) -> None:
    model, metadata = load_model(model_path, model_cfg)
    model_input = prepare_inference_input(raw_input, metadata)
    prediction = predict(model, model_input)
    save_prediction(prediction, output_dir)
```

- 학습에 사용한 feature 순서·normalizer·label inverse transform은 checkpoint metadata 또는 공용 data 함수에서 가져온다. 추론 파일에 상수로 복사하지 않는다.
- inference 전용 계산이 모델의 기존 `predict`/`rollout` 계약으로 표현되면 그 메서드를 재사용한다.
- 실측 정답이 있는 평가형 추론과 없는 운영형 추론을 구분한다. 없는 값을 억지로 metric으로 출력하지 않는다.

### 최적화 진입점: `optimize_<algorithm>.py`

순서: 설정 → 시작점·bounds·surrogate 입력 검증 → weight/model load → optimizer 생성 → 실행 → 추천값을 외부 형식으로 변환 → 결과·설정 저장.

```python
def main(model, initial_params, *, bounds: dict, search_cfg: dict,
         output_dir: str) -> None:
    optimizer = ExampleOptimizer(model=model, bounds=bounds, **search_cfg)
    rows, best = optimizer.optimize(initial_params)
    save_trials(rows, output_dir)
    save_recommendation(best, output_dir)
```

- 탐색 알고리즘의 loop·목적함수·projection·dedup은 `src/optimizer/`에 둔다.
- 실행 파일은 optimizer 반환값을 표시·저장할 수 있지만, 동일 점수를 다시 계산하지 않는다.
- 장비나 외부 시스템이 요구하는 JSON/CSV 변환은 기존 공용 converter를 사용한다.
- 탐색점수와 최종 배포점수가 다른 경우, 둘의 이름·파일·console 표기를 구분한다.

### 전처리 진입점: `preprocess_<pipeline>.py`

순서: 설정 → 입력 파일 발견·검증 → `src/data/` 변환 호출 → 누락/제외 항목 보고 → 정해진 열 순서로 저장.

```python
def main(input_dir: Path, output_path: Path, *, preprocess_cfg: dict) -> None:
    rows = []
    for source in discover_inputs(input_dir):
        row = build_row(source, preprocess_cfg)
        if row is not None:
            rows.append(row)
    pd.DataFrame(rows, columns=OUTPUT_COLUMNS).to_csv(output_path, index=False)
```

- 여러 파일을 순회하는 경우 입력 순서를 `sorted()` 등으로 고정해 재현성을 확보한다.
- 누락·제외는 `[SKIP]`, `[EXCLUDE]`처럼 이유와 대상이 보이는 한 줄 로그로 남긴다.
- 열 순서·KPI 순서·외부 schema는 `src/data/` 상수 또는 YAML에 한 번만 정의한다.

### `__main__` 구성 템플릿

```python
if __name__ == '__main__':
    cfg_path = os.environ.get('PIPELINE_CONFIG', DEFAULT_CONFIG)
    with open(cfg_path, 'r') as f:
        cfg = yaml.safe_load(f)

    model_cfg = cfg['model']
    path_cfg = cfg['path']
    train_cfg = cfg['train']
    epochs = int(train_cfg['epochs'])
    out_dir = path_cfg['output_dir']

    validate_config(cfg)
    os.makedirs(out_dir, exist_ok=True)
    with open(os.path.join(out_dir, 'config.yaml'), 'w') as f:
        yaml.safe_dump(cfg, f, allow_unicode=True, sort_keys=False)

    main(..., model_cfg=model_cfg, epochs=epochs, out_dir=out_dir)
```

- 환경변수 이름은 파이프라인을 구분한다. 예: `FORECAST_TRAIN_CONFIG`, `FORECAST_INFERENCE_CONFIG`.
- `validate_config()`은 여러 진입점에서 재사용되는 검증일 때만 만든다. 한 번 쓰는 단순 검증은 `__main__`에 둔다.
- 입력 profile의 shape·열 순서, checkpoint 존재 여부, bounds, 설정의 선택값은 실행 전에 검증한다.

## 코드 작성 규약

### 단순성

- 함수는 재사용되거나 역할·검증 경계가 분명할 때만 분리한다.
- 한 번만 호출되는 한두 줄짜리 보조 함수, 항상 같은 값인 확장 인자, 미사용 인스턴스 속성은 만들지 않는다.
- 책임이 다른 학습/추론, 탐색/최종 채점, 외부/내부 데이터 변환은 억지로 합치지 않는다.

### 이름과 데이터 표현

- 새 도메인에는 정확한 도메인 용어를 사용하고, 기하학적·일반 용어로 업무 개념을 흐리지 않는다.
- 순서가 의미 있는 차원·채널·zone 목록은 데이터 열, 모델 입력, 정규화 통계에서 같은 순서를 유지한다.
- 장비·외부 시스템과의 입출력 변환은 한 곳에 두고 실행 파일에서 직접 재조립하지 않는다.
- 여러 표현이 필요한 경우 변환은 경계에서 한 번만 한다.

### 주석과 docstring

- 주석은 **개조식**으로, 코드만으로 알 수 없는 이유·제약·불변식만 적는다.
- 서술형 문단, `in/out` 목록, 코드 줄을 그대로 번역한 주석은 쓰지 않는다.

```python
# 좋음: 제어 파라미터별 범위 차이를 없애 공통 학습률을 쓴다.
params_unit = ((start - lower) / (upper - lower)).requires_grad_(True)

# 나쁨: params_unit을 계산한다.
params_unit = ((start - lower) / (upper - lower)).requires_grad_(True)
```

- public 함수에는 한 줄 docstring을 쓴다. shape·단위·제약이 위험할 때만 짧게 덧붙인다.
- 외부 입력·파일·설정은 구체적인 예외와 문제 경로를 포함해 검증한다.

### 스타일

- import 순서: 표준 라이브러리 → 서드파티 → `src` 로컬 모듈. 각 묶음은 한 줄 공백으로 구분한다.
- 타입 힌트는 프로젝트 Python 버전과 인접 코드의 문법을 따른다.
- 미사용 import·인자·속성과 불필요한 공백 줄을 남기지 않는다.
- 순수 계산은 함수로, 상태를 오래 보유하는 모델·학습·탐색 흐름은 클래스로 둔다.

### 상태형 모듈은 클래스 하나

- `src/models/`, `src/trainer/`, `src/optimizer/`의 일반 모듈은 **핵심 public class 하나**로 구성한다.
- 해당 파일에서 쓰는 보조 로직은 class 밖 함수로 빼지 말고 `_build_*`, `_validate_*`, `_score_*` 같은 private method로 둔다.
- module 밖에는 import, 필요한 상수, type alias만 둔다. 실행 가능한 보조 함수는 두지 않는다.
- helper가 여러 클래스·모듈에서 실제로 재사용될 때는 성격에 맞는 `src/` 폴더의 독립 함수로 옮긴다.
- 하나의 알고리즘을 조금만 변형한 실험 구현은 기존 optimizer의 subclass 하나로 둔다. 별도 helper module을 만들지 않는다.

```python
# 좋음: 탐색 상태와 bounds 검증이 같은 class에 있다.
class ExampleOptimizer:
    def __init__(self, bounds: dict):
        self.bounds = self._validate_bounds(bounds)

    def _validate_bounds(self, bounds: dict) -> dict:
        if any(low > high for low, high in bounds.values()):
            raise ValueError('lower bound > upper bound')
        return bounds

# 나쁨: class 전용 보조 함수를 module 밖에 둔다.
def validate_bounds(bounds: dict) -> dict:
    ...
```

### `src/` 분류를 먼저 사용한다

- 새 코드를 만들기 전에 먼저 책임을 분류하고, 이미 있는 해당 폴더에 둔다. 편의를 위해 실행 파일이나 상태형 class module에 섞어 넣지 않는다.

| 코드 성격 | 둘 위치 | 예시 |
|---|---|---|
| 외부 파일·배열·schema 변환, 전처리, 도메인 계산 | `src/data/` | `load_data`, `json_to_params`, KPI 계산 |
| 모델 architecture·순수 tensor layer | `src/nn/` | `MLP`, `RecurrentNet` |
| 모델별 예측·학습 상태·checkpoint 계약 | `src/models/` | `GPPredictionModel` |
| 학습 수명주기·epoch/batch·checkpoint 선택 | `src/trainer/` | `ForecastTrainer` |
| 고정 surrogate 위 후보 생성·목적함수·추천 | `src/optimizer/` | `ForecastGradientDescent` |
| 여러 상위 모듈에서 쓰는 작고 도메인 독립적인 보조 기능 | `src/utils/` | seed, path/bounds 해석 |
| 결과를 입력으로 받아 그림을 만드는 함수 | `src/visualization/` | `plot_curve` |

- 판단 순서:
  1. 해당 로직이 model/trainer/optimizer **한 class의 상태**를 쓰는가? → 그 class의 private method.
  2. 데이터 형식·도메인 규칙을 아는가? → `src/data/`.
  3. 여러 계층이 재사용하는 작고 독립적인 계산인가? → `src/utils/`.
  4. 모델 layer인가? → `src/nn/`; 모델 lifecycle 계약인가? → `src/models/`.
  5. 결과를 그리는가? → `src/visualization/`.

- 새 `helpers.py`, `common.py`, `misc.py` 같은 모호한 덤프 파일은 만들지 않는다.
- 하나의 함수만을 위해 새 파일을 만들지 않는다. 같은 책임의 기존 파일에 추가하거나, 책임이 충분히 독립적이고 재사용될 때만 새 파일을 만든다.

## 명명 규약

이름만 보고 역할·단위·범위를 짐작할 수 있어야 한다. 기존 파일의 import 경로와 checkpoint key는 계약이므로, 정리 목적의 rename은 사용자 요청이 있을 때만 한다.

### 파일·class·public method 양식

| 위치 | 파일명 양식 | class 양식 | public method 양식 |
|---|---|---|---|
| `src/nn/` | `<Architecture>.py` | `<Architecture>` 또는 `<Architecture>Net` | `forward`, 필요 시 `step` |
| `src/models/` | `<Architecture>_prediction_model.py` | `<Architecture>PredictionModel` | `forward`/`predict`, `update`/`fit`, `evaluate`, 필요 시 `rollout` |
| `src/trainer/` | `<Pipeline>_trainer.py` | `<Pipeline>Trainer` | `train`, `log_epoch`, `evaluate`, 필요한 명시적 metric |
| `src/optimizer/` | `<domain>_<algorithm>.py` | `<Domain><Algorithm>` | `optimize` 또는 알고리즘명 `random_search` |
| `src/data/` | `<domain>_<responsibility>.py` | class가 필요 없으면 함수 중심 | `load_*`, `build_*`, `calculate_*`, `normalize_*` |
| `src/utils/` | `<responsibility>.py` | class가 필요 없으면 함수 중심 | `fix_*`, `resolve_*`, `format_*` |
| `src/visualization/` | `plot_<subject>.py` | 보통 class 없음 | `plot_<subject>` |

- 기존 양식 사례:
  - `src/nn/MLP.py` → `MLP`, `src/nn/Recurrent.py` → `RecurrentNet`
  - `src/models/GRU_prediction_model.py` → `GRUPredictionModel`
  - `src/trainer/forecast_trainer.py` → `ForecastTrainer`
  - `src/optimizer/forecast_gradient_descent.py` → `ForecastGradientDescent`
- 새 예측 모델은 `<Architecture>_prediction_model.py` / `<Architecture>PredictionModel`을 따른다.
- 새 파이프라인 trainer는 `<Pipeline>_trainer.py` / `<Pipeline>Trainer`를 따른다. 기존 generic `Trainer`는 legacy 선례이며 새 코드의 이름으로 쓰지 않는다.
- 새 optimizer는 대상과 알고리즘을 모두 드러낸다. `optimizer.py`, `search.py`, `utils.py`처럼 의미가 넓은 파일명은 쓰지 않는다.
- public method는 동사로 시작한다. private method는 `_validate_*`, `_build_*`, `_predict_*`, `_score_*`, `_record_*`처럼 내부 단계를 드러낸다.

### 변수·인자·반환값 양식

- 설정 묶음: `model_cfg`, `trainer_cfg`, `path_cfg`, `search_cfg`, `data_cfg`.
- 경로: 파일은 `*_path`, 디렉터리는 `*_dir`, checkpoint는 `checkpoint_path`, 출력은 `output_dir`.
- 데이터: 역할과 범위를 함께 쓴다. `raw_data`, `train_data`, `validation_data`, `test_data`, `feature_cols`, `target_values`, `prediction_rows`.
- 모델·학습: `model`, `trainer`, `optimizer`, `learning_rate`, `batch_size`, `epoch`, `best_metric`, `best_model_state`.
- 탐색: `control_params`, `initial_params`, `candidate_params`, `bounds`, `objective_value`, `best_result`, `trial_rows`.
- 도메인 파라미터에는 해당 업무의 정확한 이름을 쓴다.
- 단위가 있는 값은 이름에 단위 또는 기준을 드러낸다. 예: `temperature_c`, `elapsed_seconds`, `rmse_c`, `normalized_loss`.
- boolean은 `is_`, `has_`, `should_`, `use_`로 시작한다. 예: `has_checkpoint`, `use_quantization`.
- collection은 복수형, 단일 값은 단수형으로 쓴다. 예: `test_runs` / `test_run`, `rows` / `row`.
- 한 글자 변수는 수식·tensor 연산의 짧은 지역 범위 또는 단순 loop index에서만 쓴다. public 인자와 여러 단계에 걸쳐 유지되는 변수에는 `m`, `p`, `s`, `x`, `r` 같은 축약을 쓰지 않는다.
- 약어는 공식 모델·지표명에만 쓴다. `MLP`, `GRU`, `KPI`, `RMSE`는 허용하지만 Python 이름은 `kpi_values`, `rmse_c`처럼 읽히게 쓴다.

```python
# 좋음
model_cfg = cfg['model']
checkpoint_path = os.path.join(model_dir, model_name)
candidate_params = optimizer.sample_candidate()
objective_value = result['J']

# 나쁨: 범위와 역할을 알 수 없다.
m, p, s = cfg['model'], cfg['path'], cfg['search']
x = opt.run(p)
```

## 레이어별 코드 스타일

### `src/nn/`: PyTorch 순수 네트워크 블록

- PyTorch 모델일 때만 사용한다. `nn.Module` 외 파일 I/O, YAML, 데이터 분할, 학습 loop, checkpoint 저장을 넣지 않는다.
- `__init__()`은 layer·activation·차원만 구성하고, `forward()`는 tensor 계산만 한다.
- 모델군이 같은 호출 계약을 공유해야 하면 `forward()`와 필요한 `step()`의 입력·출력을 맞춘다.
- 모델 종류 선택은 실제로 같은 생성자 계약을 공유할 때만 `NETS` 같은 작은 mapping으로 둔다.

```python
class Encoder(nn.Module):
    def __init__(self, input_dim: int, hidden_dim: int, output_dim: int):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, output_dim),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.net(x)
```

- 새 네트워크가 hidden state, sequence, multi-input처럼 다른 계약을 가져도 기존 MLP처럼 보이게 억지로 맞추지 않는다. 그 계약이 필요한 모델 wrapper에서 변환한다.

### `src/models/`: 모델 계약과 상태

- 모델 파일은 보통 모델 하나와 그 public 계약을 둔다. 이름은 `<Model>PredictionModel`처럼 역할이 드러나게 짓는다.
- 모델은 내부 estimator, 모델 고유 상태, 예측·학습·평가 계약을 가진다. PyTorch 모델에서는 네트워크·buffer·`forward()`가 이에 해당한다.
- 학습을 `model.update()`로 위임하는 파이프라인에서는 optimizer와 한 배치 갱신도 모델이 소유한다. trainer가 loss 수식을 복제하지 않는다.
- 반대로 다른 파이프라인이 trainer 소유 optimizer를 이미 사용한다면 그 계약을 따르고, 한 파이프라인 안에서 두 방식을 섞지 않는다.
- PyTorch checkpoint에 필요한 평균·표준편차·상수 tensor는 일반 속성이 아니라 `register_buffer()`로 등록한다.

```python
class ExamplePredictionModel(nn.Module):
    def __init__(self, normalizer: dict, device: str):
        super().__init__()
        self.net = Encoder(...)
        self.register_buffer('x_mean', torch.as_tensor(normalizer['x_mean']))
        self.device = torch.device(device if torch.cuda.is_available() else 'cpu')
        self.to(self.device)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.net((x - self.x_mean) / self.x_std)
```

- public 메서드는 호출 흐름이 보이게 둔다. 예: `predict` 또는 `forward` → 필요한 sequence 처리 → `update`/`fit` → `evaluate`.
- 구 checkpoint 호환 처리는 해당 load 메서드에 국소화한다. PyTorch라면 `load_state_dict()`를 쓴다.
- 새 모델이 기존 모델군과 같은 trainer를 쓸 때는 trainer가 호출하는 메서드명·반환 shape·반환 dict 키를 맞춘다. 다르면 새 trainer 또는 명시적인 adapter를 만든다.

### `src/trainer/`: 학습 수명주기

- trainer는 epoch/batch loop, batch 순서, scheduler, best checkpoint, 학습 로그, test 평가를 담당한다.
- trainer는 내부 network를 직접 호출하거나 도메인 feature를 재구현하지 않는다. `model.update()`, `model.fit()`, `model.evaluate()` 같은 모델 계약을 호출한다.
- `train()`은 학습 이력·최고 점수·테스트 결과처럼 호출자가 저장·보고할 값을 반환한다.
- epoch별 CSV 열은 `log_epoch()`처럼 한 곳에서 조립하고, console log와 파일의 책임을 분리한다.
- 긴 학습은 중간 결과를 남기고, best와 last checkpoint를 구분한다.

```python
# PyTorch batch 모델의 예시
for epoch in range(self.epochs):
    model.train()
    batch_stats = [model.update(xb, yb) for xb, yb in batches]
    with torch.no_grad():
        val_loss = model.evaluate(validation_data)
    history.append(self.log_epoch(epoch, batch_stats, val_loss))
```

- PyTorch 평가의 `torch.no_grad()`는 평가 진입점에서 한 번 감싼다. 다른 framework에는 그 framework의 평가 문맥을 쓴다.

### `src/optimizer/`: 고정 모델 위 후보 탐색

- optimizer는 학습된 model/surrogate를 예측 전용 상태로 두고 후보와 목적함수만 최적화한다. PyTorch surrogate라면 `eval()`과 parameter freeze를 적용한다.
- public 진입점은 `optimize()` 또는 알고리즘을 드러내는 `random_search()` 하나로 둔다. 내부 계산은 `_predict_*`, `_acquisition_*` 같은 private helper로 숨긴다.
- bounds 검증·정규화·projection은 후보 생성 직후 또는 update 직후 한 곳에서 처리한다.
- 탐색 결과는 후보 값, 목적함수 값, 필요한 KPI/진단값을 한 행 dict로 기록한다. 실행 파일은 그 기록을 다시 계산하지 않는다.
- optimizer에 모델 학습 loop, checkpoint 저장, 외부 입력 파싱을 넣지 않는다.

```python
def optimize(self, n_step: int) -> tuple[list[dict], dict]:
    rows = []
    for step in range(n_step):
        candidate = self._next_candidate()
        score = self._score(candidate)
        rows.append({'step': step, **candidate, **score})
    return rows, min(rows, key=lambda row: row['J'])
```

- 탐색 목적함수와 배포·장비 기준 최종 채점이 다르면 public 메서드 둘로 분리하고, 차이를 이름과 docstring에 밝힌다.

## 클래스 구성 템플릿

새 클래스는 먼저 framework와 무관한 public 계약을 정하고, 아래 사례 중 **데이터·학습 계약이 같은 것**만 선택한다. 이름만 바꿔 복사하지 말고, 필요 없는 메서드는 만들지 않는다.

### 공통 모델 계약

| 상황 | 최소 public 메서드 | 반환 규약 |
|---|---|---|
| batch 학습 모델 | `update` 또는 `fit`, `evaluate`, `predict` | loss/metric 또는 예측값 |
| 확률 예측 모델 | `update` 또는 `fit`, `predict`, `evaluate`, `save/load` | mean·uncertainty·metric·복원 metadata |
| sequence 모델 | 위 계약 + 필요한 `step`/`rollout` | 모델군과 같은 shape·state 규약 |
| 학습 없는 surrogate | `predict`/`evaluate`만 | optimizer가 소비하는 점수 또는 분포 |

- `forward`, `register_buffer`, `state_dict`는 PyTorch의 선택적 세부 구현이다. 모든 새 모델의 필수 메서드가 아니다.
- trainer가 기대하는 메서드·인자·반환값을 먼저 문서화하고, 모델의 framework별 API를 그 경계 뒤에 숨긴다.

### 1. PyTorch 예측 모델 사례: `src/models/<model>_prediction_model.py`

trainer가 `model.update()`를 호출하는 모델의 구성이다.

```python
class ExamplePredictionModel(nn.Module):
    def __init__(self, input_spec, normalizer: dict, optimizer: str,
                 lr: float, device: str):
        super().__init__()
        # - 입력 차원과 순서를 확정한다.
        self.net = ExampleNet(...)
        # - checkpoint에 필요한 통계를 buffer로 보관한다.
        self.register_buffer('x_mean', torch.as_tensor(normalizer['x_mean']))
        self.register_buffer('x_std', torch.as_tensor(normalizer['x_std']))
        self.device = torch.device(device if torch.cuda.is_available() else 'cpu')
        self.to(self.device)
        self.optimizer = getattr(torch.optim, optimizer)(self.parameters(), lr=lr)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.net((x - self.x_mean) / self.x_std)

    def update(self, xb, yb) -> dict:
        self.optimizer.zero_grad()
        loss = self.loss_fn(self(xb), yb)
        loss.backward()
        self.optimizer.step()
        return {'loss': float(loss), 'n': len(xb)}

    def evaluate(self, data) -> float:
        ...
```

- `__init__`: 입력 차원·네트워크·정규화 buffer·device·학습 상태를 구성한다.
- `forward`: 입력 tensor를 모델 출력으로만 변환한다. 파일 저장, metric 출력, `no_grad()`를 넣지 않는다.
- `update`: 한 batch의 zero-grad → forward/loss → backward → step을 수행하고 trainer가 집계할 dict를 반환한다.
- `evaluate`: trainer가 검증 시 호출하는 단일 scalar 또는 명시적 metric dict를 반환한다.
- autoregressive·sequence 모델처럼 필요할 때만 `make_frame` → `step` → `next_y` → `rollout_loss` → `rollout` 순으로 추가한다.
- 새 모델이 기존 trainer를 재사용하면 그 trainer가 요구하는 입력·상태·public method와 반환 계약을 모두 만족해야 한다. 하나라도 다르면 별도 trainer를 만든다.

### 2. 확률 모델 wrapper 사례: `src/models/<model>_prediction_model.py`

GP처럼 underlying model 외에 likelihood, objective, 저장 metadata가 모델의 일부인 경우의 구성이다.

```python
class ExamplePredictionModel:
    def __init__(self, train_x, train_y, device: str, lr: float, **model_cfg):
        self.device = select_device(device)
        self.model = ExampleProbabilisticModel(train_x, train_y, **model_cfg).to(self.device)
        self.likelihood = build_likelihood(...).to(self.device)
        self.optimizer = torch.optim.Adam(self.model.parameters(), lr=lr)
        self.objective = build_objective(self.model, self.likelihood)

    def update(self, train_x, train_y) -> float: ...
    def predict(self, x): ...
    def evaluate(self, x, y) -> dict: ...
    def state_dict(self) -> dict: ...
    def load_state_dict(self, state: dict) -> None: ...
    def save(self, path, ..., metadata) -> None: ...
```

- wrapper의 `state_dict`/`load_state_dict`는 내부 model과 likelihood 등 복원에 필요한 상태를 한 계약으로 묶는다.
- `save()`는 모델 state뿐 아니라 inference·optimizer가 필요한 feature 순서, 정규화 통계, target metadata를 같이 저장한다.
- `predict()`는 eval/no-grad와 출력의 device→numpy 변환을 한 곳에서 책임진다.
- trainer는 이 wrapper의 `update/predict/evaluate`만 호출하고, 내부 likelihood나 objective에 직접 접근하지 않는다.

### 3. Trainer: `src/trainer/<pipeline>_trainer.py`

Trainer는 파이프라인별 학습 수명주기를 표현한다. 모델군마다 trainer가 다를 수 있는 이유는 batch 방식·검증 방식이 다르기 때문이다.

```python
# PyTorch epoch 모델의 예시
class ExampleTrainer:
    def __init__(self, epochs: int, batch_size: int, out_dir, model_dir,
                 seed: int, verbose: bool = True):
        # - YAML 값을 경계에서 형변환한다.
        self.epochs = int(epochs)
        self.batch_size = int(batch_size)
        self.out_dir = Path(out_dir)
        self.model_dir = Path(model_dir)
        self.rng = np.random.RandomState(seed)
        self.out_dir.mkdir(parents=True, exist_ok=True)
        self.model_dir.mkdir(parents=True, exist_ok=True)

    def train(self, model, train_data, validation_data, test_data) -> dict:
        best_metric, best_state, history = float('inf'), None, {}
        for epoch in range(self.epochs):
            batch_stats = self._train_epoch(model, train_data)
            with torch.no_grad():
                val_metric = model.evaluate(validation_data)
            row = self.log_epoch(epoch, batch_stats, val_metric)
            self._append_history(history, row)
            if val_metric < best_metric:
                best_metric, best_state = val_metric, copy.deepcopy(model.state_dict())
                torch.save(best_state, self.model_dir / 'best.pt')
        model.load_state_dict(best_state)
        test_rows = self.evaluate(model, test_data)
        return {'history': history, 'best_metric': best_metric, 'test_rows': test_rows}

    def log_epoch(self, epoch: int, batch_stats: list[dict], val_metric: float) -> dict: ...
    def evaluate(self, model, test_data) -> list[dict]: ...
```

- `__init__`: YAML-derived run configuration, output directories, random state만 보관한다. 모델·데이터를 생성하지 않는다.
- `train`: 학습 loop, scheduler, best/last checkpoint, 최종 test 평가의 단일 진입점이다.
- `log_epoch`: CSV 한 행의 열 순서·집계 규칙·console log를 한 곳에서 관리한다.
- `evaluate`: test 단위마다 하나의 결과 dict를 만들고, 반복 사용될 metric은 별도 public helper로 분리한다. 예: `nstep_rmse()`.
- K-fold가 핵심 평가라면 `train()`과 `evaluate_kfold_cv()`를 별도 public 메서드로 둔다. batch loop 형태로 억지로 맞추지 않는다.
- trainer가 model의 내부 network, loss 수식, input feature 조립을 복제하면 안 된다.

### 4. Optimizer: `src/optimizer/<algorithm>.py`

```python
# 미분 가능한 surrogate의 예시
class ExampleOptimizer:
    def __init__(self, model, bounds: dict, objective_spec: dict, device: str):
        self.model = model
        self.bounds = self._validate_bounds(bounds)
        self.objective_spec = objective_spec
        self.device = select_device(device)
        self.model.eval()
        for parameter in self.model.parameters():
            parameter.requires_grad_(False)

    def optimize(self, n_trial: int) -> tuple[list[dict], dict]:
        rows = []
        for trial in range(1, n_trial + 1):
            candidate = self._sample_candidate()
            result = self._score(candidate)
            rows.append({'trial': trial, **candidate, **result})
        return rows, min(rows, key=lambda row: row['J'])

    def _score(self, candidate) -> dict: ...
```

- `__init__`: model/surrogate 복원 또는 freeze, bounds·목적함수 검증, 후보 표현과 사전 계산을 수행한다.
- public method: 알고리즘의 실행 하나만 노출한다. `optimize`, `random_search`처럼 호출 의도가 드러나는 이름을 쓴다.
- private helper: `_predict_*` → `_acquisition_*` → `_candidate_*` → `_record_*`처럼 계산 단계별로 묶는다.
- `optimize()`는 후보 생성 → score → bounds projection/dedup → record → best 선택 순서를 보이게 유지한다.
- optimizer의 return은 실행 파일이 바로 저장·표시할 수 있는 결과여야 한다. 실행 파일이 objective를 재계산하지 않는다.
- 실험 전용 파생 optimizer는 기존 알고리즘의 작은 부분만 다를 때만 상속한다. 예: 기존 KPI 예측만 대체하는 `MultiObjectiveBayesianOptimizerExp1`.

## 구현쌍과 모델군

- 기준 구현과 미분 가능 구현, CPU/GPU 구현, 또는 모델군이 있을 때만 대응 관계를 명시한다.
- 대응 구현은 다음을 함께 변경·검증한다.
  - 입력·출력 계약
  - 수식·양자화·경계 조건
  - 공개 함수 이름과 인자 순서
  - 비교 테스트 또는 동등성 검증
- numpy/tensor 재현식과 MLP/RNN/GRU/LSTM은 이 원칙을 적용할 수 있는 사례다. 새 모델이 이 구조를 따를 필요는 없다.
- 공통 경로가 이미 있다면 학습·평가·추론에서 재사용한다. 새 경로는 기존 계약으로 표현할 수 없을 때만 추가한다.

## 산출물 규약

- 저장 형식은 소비자에 맞춘다. 표·학습 곡선은 CSV, 구조화된 외부 입력은 JSON, 실행 재현 정보는 YAML이 기본 선례다.
- 새 산출물은 `results/<pipeline>/`, `figs/<pipeline>/`, `saved_models/<pipeline>/`처럼 파이프라인 경계를 드러내는 위치에 둔다.
- 같은 결과를 두 번 저장하지 않는다. 저장 책임은 실행 파일·trainer·optimizer 중 한 곳만 가진다.
- 새 산출물이나 디렉터리를 추가하면 관련 README에 위치와 목적을 적는다.

## 변경 전 체크리스트

- [ ] 템플릿화 기준 베이스라인(notebook 또는 `.py`)과 연결된 config·데이터·checkpoint를 확정했는가?
- [ ] 목적함수·평가 구간·seed·후처리 등 성능을 바꾸는 세부 의도를 확인했는가?
- [ ] baseline과 변경 후의 동일성 metric·허용 오차·비교 방법을 정했는가?
- [ ] 새 파일의 책임이 폴더 역할과 맞는가?
- [ ] 기존 기능과 계약을 공유하는지, 독립 파이프라인인지 판단했는가?
- [ ] YAML 키·경로·출력 형식을 한 곳에서만 정의했는가?
- [ ] 대응 구현 또는 모델군이 있다면 함께 확인했는가?
- [ ] 새 함수·인자·클래스가 실제 필요하며 호출되는가?
- [ ] 주석이 개조식이고 이유만 설명하는가?
- [ ] 관련 README와 문법 검사를 갱신·실행했는가?
