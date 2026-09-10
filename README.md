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

### Codex에서 사용

Codex 안에서는 내장 설치 도구를 호출해 GitHub 저장소에서 설치할 수 있습니다.

```text
$skill-installer SungsooKwon/ai-template-skill
```

Codex 전역 스킬로 설치하면 모든 프로젝트에서 사용할 수 있습니다.

```bash
git clone https://github.com/SungsooKwon/ai-template-skill.git /tmp/ai-template-skill
mkdir -p ~/.agents/skills
cp -R /tmp/ai-template-skill/python-ml-pipeline-conventions ~/.agents/skills/
```

특정 프로젝트에서만 사용할 때는 해당 프로젝트 루트에 설치합니다.

```bash
git clone https://github.com/SungsooKwon/ai-template-skill.git /tmp/ai-template-skill
mkdir -p .agents/skills
cp -R /tmp/ai-template-skill/python-ml-pipeline-conventions .agents/skills/
```

Codex는 작업이 스킬 설명과 일치하면 자동으로 선택할 수 있습니다. 확실히 적용하려면 프롬프트에 `$python-ml-pipeline-conventions`를 넣습니다.

```text
$python-ml-pipeline-conventions 새 예측 모델의 학습 파이프라인을 추가해줘.
```

설치 또는 업데이트가 보이지 않으면 Codex를 재시작합니다.

### Claude Code에서 사용

Claude Code를 사용하는 프로젝트라면 기존처럼 `.claude/skills`에 복사할 수 있습니다.

```bash
git clone https://github.com/SungsooKwon/ai-template-skill.git /tmp/ai-template-skill
mkdir -p .claude/skills
cp -R /tmp/ai-template-skill/python-ml-pipeline-conventions .claude/skills/
```

## 커스터마이즈

스킬은 공통 원칙을 제공합니다. 프로젝트 고유의 데이터 schema, 실행 명령, 외부 시스템 계약, 도메인 용어는 각 프로젝트의 별도 스킬 또는 `SKILL.md` 보완 규칙에 추가하세요.

## License

필요에 맞게 사용·수정하세요.
