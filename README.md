# AI Template Skills

재사용 가능한 AI 코딩 에이전트 스킬 모음입니다. 각 스킬은 프로젝트에 복사해 에이전트가 코드 구조와 작업 원칙을 일관되게 따르도록 돕습니다.

## Included skills

### `python-ml-pipeline-conventions`

Python 기반 머신러닝 모델·파이프라인을 위한 구현 규약입니다.

- `root entrypoint → src 역할별 모듈 → configs → 산출물` 구조
- YAML을 실행 설정의 단일 출처로 유지
- 모델, trainer, optimizer, data, visualization의 책임 분리
- 필요 전의 과도한 추상화와 범용 프레임워크 도입 방지
- 템플릿화·리팩터링 전후 베이스라인 재현성 검증

`Hydra`, 실험 관리 프레임워크, CLI 파서 없이 가볍고 직선적인 Python/PyTorch 프로젝트를 구성할 때 적합합니다.

## 사용 방법

원하는 프로젝트의 스킬 디렉터리에 복사합니다.

```bash
mkdir -p .claude/skills
cp -R python-ml-pipeline-conventions .claude/skills/
```

그 뒤 새 모델, 학습·추론·최적화 파이프라인, YAML 설정, 산출물을 추가하거나 수정하는 작업에서 이 스킬을 사용하면 됩니다.

## 커스터마이즈

스킬은 공통 원칙을 제공합니다. 프로젝트 고유의 데이터 schema, 실행 명령, 외부 시스템 계약, 도메인 용어는 각 프로젝트의 별도 스킬 또는 `SKILL.md` 보완 규칙에 추가하세요.

## License

필요에 맞게 사용·수정하세요.
