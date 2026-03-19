# MiniMax M2.7의 Self-Evolution: 가중치는 안 바뀌는데 능력은 어떻게 익혀질까?

## 1. 들어가며

"첫 번째로 자기 진화에 깊게 참여한 모델" - MiniMax M2.7의 소개 문구를 보면 뭔가 특별할 것 같은 느낌이 듭니다. 하지만 "Self-Evolution"이라는 단어는 동시에 의문을 자아냅니다. **"가중치가 안 바뀌는데 능력이 익혀진다? 대체 무슨 마법일까?"**

이 글에서는 MiniMax M2.7이 사용하는 Self-Evolution 방식을 현존하는 다른 접근방식과 비교하며, 그 핵심 메커니즘을 파헤쳐 보겠습니다.

---

## 2. 기존 접근방식 vs M2.7의 Self-Evolution

### 2.1 전통적인 LLM 학습

전통적인 LLM 학습은 아주 직관적입니다:

```python
# 전통적인 fine-tuning (개념적 예시)
for epoch in epochs:
    for batch in data:
        loss = model(batch) - target
        gradients = compute_gradients(loss)
        weights = weights - learning_rate * gradients
```

**핵심:**
- 가중치를 직접 업데이트합니다
- 한 번 학습하면 모델의 능력이 가중치에 고정됩니다
- 새로운 능력을 익히려면 새로운 학습 데이터와 재학습이 필요합니다

### 2.2 M2.7의 Self-Evolution

MiniMax M2.7은 다른 접근을 취합니다:

```python
# M2.7의 Self-Evolution (개념적 예시)
for iteration in range(100):  # 100회 이상 반복
    # 1. 실패 분석
    failures = analyze_failure_trajectories()

    # 2. 개선 계획 수립
    plan = plan_changes(failures)

    # 3. Scaffold 코드 수정
    modify_scaffold_code(plan)

    # 4. 평가 실행
    results = run_evaluations()

    # 5. 결과 비교 및 결정
    if results.better():
        keep_changes()
    else:
        revert_changes()
```

**핵심 차이점:**
- M2.7 자신의 가중치는 바뀌지 않습니다
- 대신 M2.7이 **agent로서** 자신을 둘러싼 환경(scaffold, tools, memory)을 개선합니다
- 이 과정에서 생성된 **고품질 데이터**가 최종 모델 학습에 사용됩니다

---

## 3. Self-Evolution 과정: 100회 이상의 자기 개선 루프

MiniMax는 내부 M2.7 모델을 사용하여 연구 agent harness를 구축했습니다. 이 harness는 다음과 같은 능력을 갖추고 있습니다:

- 데이터 파이프라인 관리
- 학습 환경 설정
- 인프라 관리
- 팀 간 협업 지원
- 지속적인 메모리 유지

### 3.1 실제 사용 사례: RL 팀의 일상

RL 팀의 일상적인 업무 흐름을 예로 들어보겠습니다:

```python
# 연구자가 M2.7 agent와 협업하는 과정

# 1. 아이디어 논의
researcher.discuss_idea_with_agent()

# 2. 문헌 조사
agent.literature_review()

# 3. 실험 사양 설정
agent.set_experiment_spec()

# 4. 데이터 파이프라인 구축
agent.pipeline_data()

# 5. 실험 실행
agent.launch_experiments()

# 6. 실험 중 모니터링
while experiment.running:
    agent.monitor_progress()
    agent.read_logs()
    agent.debug_issues()
    agent.analyze_metrics()
    agent.fix_code()
    agent.submit_merge_request()
    agent.run_smoke_tests()
```

이 과정에서 **인간 연구자는 오직 중요한 의사결정과 토론에만 참여**하고, 나머지 30-50%의 워크플로우는 M2.7이 자동으로 처리합니다.

### 3.2 자기 진화 루프: 100회 이상의 반복

가장 인상적인 부분은 M2.7이 자신의 scaffold를 최적화한 사례입니다:

```
반복 1: 실패 분석 → temperature=0.7 시도 → 평가 → 결과 기록
반복 2: 실패 분석 → temperature=0.5 시도 → 평가 → 결과 기록
반복 3: 실패 분석 → frequency_penalty 추가 → 평가 → 결과 기록
...
반복 50: 패턴 발견 → "같은 버그 패턴 다른 파일에서 검색" 규칙 추가
반복 100: 최종 조합 도달 → 30% 성능 향상 달성
```

이 과정에서 M2.7은 다음과 같은 최적화를 자율적으로 발견했습니다:

- **샘플링 파라미터 최적화**: temperature, frequency_penalty, presence_penalty의 최적 조합
- **워크플로우 가이드라인 설계**: 예를 들어 "버그 수정 후 다른 파일에서 같은 패턴 검색"
- **루프 감지 최적화**: agent 루프에 무한 루프 방지 메커니즘 추가

결과: **내부 평가 세트에서 30% 성능 향상**

---

## 4. 가중치 핵심: "안 바뀌는데 능력이 익혀진다"의 역설

이제 가장 중요한 질문에 답해봅시다: **"도대체 가중치는 안 바뀌는데 능력은 어떻게 익혀진 거지?"**

### 4.1 두 단계로 이해하기

M2.7의 Self-Evolution은 두 개의 명확한 단계로 나뉩니다:

#### **단계 1: 개발 단계 (Development Phase)**

```
내부 M2.7 (agent) → 100회+ 자기 개선 루프 → 고품질 데이터 생성
                                                        ↓
                                              최종 M2.7 모델 학습
```

이 단계에서:
- 내부 M2.7은 **가중치가 고정된 상태**로 agent로서 작동합니다
- 100회 이상의 반복 루프를 통해 실패-성공 패턴을 학습합니다
- 이 과정에서 **수많은 시행착오 데이터**가 생성됩니다
- 이 데이터는 "이런 상황에서는 이런 방식으로 접근하면 성공한다"는 패턴을 포함합니다

#### **단계 2: 학습 단계 (Training Phase)**

```
고품질 데이터 → 최종 M2.7 가중치 학습 → 능력이 가중치에 각인됨
```

이 단계에서:
- 단계 1에서 생성된 고품질 데이터로 최종 모델을 학습합니다
- **agent가 발견한 최적의 접근방식 패턴**이 모델의 가중치에 각인됩니다
- 이제 모델은 추론 시 이 패턴을 자동으로 적용할 수 있습니다

### 4.2 비유로 이해하기

이것은 **요리사와 레시피북**의 관계와 비슷합니다:

```
개발 단계:
요리사(내부 M2.7) → 100번 요리 시도 → 성공 레시피 100개 발견
                                               ↓
                                    레시피북(데이터) 작성

학습 단계:
레시피북 → 새로운 요리사(최종 M2.7) 훈련 → 레시피 내재화

추론 단계:
새로운 요리사 → 레시피 자동 적용 → 완성된 요리
```

### 4.3 추론 시: 가중치 고정, 패턴 적용

최종 모델이 배포된 후:

```
추론 시:
사용자 요청 → M2.7 (가중치 고정) → 익힌 패턴 자동 적용 → 최적의 결과
```

이제 M2.7은:
- 가중치는 더 이상 바뀌지 않습니다
- 하지만 가중치에 **이미 최적의 접근방식 패턴이 각인**되어 있습니다
- 새로운 문제를 만나도 학습된 패턴을 자동으로 적용합니다

---

## 5. 벤치마크 성능

이런 Self-Evolution 과정을 거친 M2.7의 성능은 어떨까요?

### 5.1 소프트웨어 엔지니어링

| 벤치마크 | M2.7 | 경쟁 모델 |
|---------|------|---------|
| SWE-Pro | 56.22% | GPT-5.3-Codex와 동급 |
| VIBE-Pro | 55.6% | Opus 4.6과 거의 동급 |
| SWE Multilingual | 76.5% | 최상위권 |
| Multi SWE Bench | 52.7% | 최상위권 |
| Terminal Bench 2 | 57.0% | 최상위권 |
| NL2Repo | 39.8% | 최상위권 |

### 5.2 오피스 워크

| 벤치마크 | M2.7 | 경쟁 모델 |
|---------|------|---------|
| GDPval-AA (ELO) | 1495 | 오픈소스 모델 중 최고 |
| Toolathon | 46.3% | 글로벌 최상위권 |

### 5.3 머신러닝 경진대회 (MLE Bench Lite)

MiniMax는 OpenAI가 공개한 MLE Bench Lite 수준의 22개 머신러닝 경진대회에 M2.7을 참여시켰습니다:

- **최고 실행**: 금메달 9개, 은메달 5개, 동메달 1개
- **평균 메달 획득률**: 66.6%
- **비교**:
  - Opus-4.6: 75.7%
  - GPT-5.4: 71.2%
  - **M2.7: 66.6%**
  - Gemini-3.1: 66.6%

---

## 6. 다른 접근방식과의 비교

### 6.1 OpenAI o1/o3 계열

OpenAI의 o1/o3 모델들은 **chain-of-thought reasoning**에 집중합니다:

```python
# OpenAI o1/o3 방식 (개념적)
response = model.think_deeply(
    query,
    reasoning_steps="extended"  # 긴 사고 과정
)
```

**특징:**
- 추론 시간을 더 소비하여 더 깊은 reasoning을 수행
- 복잡한 수학, 과학 문제에 특화
- 하지만 학습 방식 자체는 전통적인 fine-tuning 기반

### 6.2 Meta의 접근방식

Meta는 주로 **RLHF (Reinforcement Learning from Human Feedback)**와 같은 전통적인 접근방식을 사용합니다:

```python
# Meta의 RLHF 방식 (개념적)
for iteration in range(num_iterations):
    # 1. 모델이 응답 생성
    responses = model.generate(prompts)

    # 2. 인간이 응답 평가
    rankings = human_rank(responses)

    # 3. 보상 모델 학습
    reward_model.train(rankings)

    # 4. 보상 모델로 기본 모델 fine-tuning
    model.fine_tune(reward_model)
```

**특징:**
- 인간 피드백을 통해 모델을 개선
- 가중치를 직접 업데이트
- 하지만 인간 피드백에 의존하므로 확장성에 한계

### 6.3 MiniMax M2.7의 차별점

```python
# MiniMax M2.7 방식 (개념적)
# 단계 1: 개발
for iteration in range(100):
    improvements = agent.self_evolve()
    high_quality_data.collect(improvements)

# 단계 2: 학습
final_model.train(high_quality_data)

# 단계 3: 추론
result = final_model.apply_learned_patterns(query)
```

**차별점:**
1. **자기 개선 루프**: 인간 개입 없이 100회 이상의 자율적 개선
2. **고품질 데이터 생성**: 실제 시행착오 과정에서 얻은 데이터
3. **패턴 내재화**: agent가 발견한 최적의 접근방식이 가중치에 각인
4. **실용성 중심**: 실제 소프트웨어 엔지니어링, 오피스 워크에 특화

---

## 7. 마무리: 왜 이것이 중요할까?

MiniMax M2.7의 Self-Evolution은 **"가중치가 안 바뀌는데 능력이 익혀진다"**는 역설을 아름답게 해결합니다:

1. **개발 단계**: 가중치를 고정한 상태로 agent로서 자기 개선 → 고품질 데이터 생성
2. **학습 단계**: 이 데이터로 최종 모델 학습 → 패턴이 가중치에 각인됨
3. **추론 단계**: 가중치는 고정, 하지만 익힌 패턴 자동 적용

이 접근방식의 중요성은 **"모델이 자기 자신의 개선 과정에 참여한다"**는 점입니다. 이것은 단순한 기술적 진보가 아니라, AI가 자신의 능력을 스스로 확장할 수 있는 가능성을 보여줍니다.

MiniMax는 **"미래의 AI 자기 진화는 점차 완전 자율성으로 나아갈 것"**이라고 말합니다. 데이터 구축, 모델 학습, 추론 아키텍처, 평가 등 모든 단계를 인간 개입 없이 조정하는 것입니다.

M2.7은 그 첫 번째 걸음일지도 모릅니다.

---

## 참고 자료

- [MiniMax M2.7 Official Announcement](https://www.minimax.io/news/minimax-m27-en)
- MLE Bench Lite: OpenAI가 공개한 머신러닝 경진대회 벤치마크
