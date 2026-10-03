아래처럼 README/실험 노트용 마크다운으로 정리하면 돼.

# Mem0-style ADD Gate 3-Way Pipeline 비교

## 1. 실험 목적

멀티턴 대화에서 새로운 long-term memory를 저장해야 하는지 판단하는 방식을 3가지 pipeline으로 비교하였다.

비교한 방법은 다음과 같다.

1. **P1: Dialogue → Retrieval → Binary**
   - 멀티턴 dialogue 전체로 existing memory를 retrieval
   - retrieved memory와 dialogue를 기반으로 LLM이 `ADD / NO_ADD` binary routing 수행

2. **P2: Dialogue → Retrieval → Fact Extraction**
   - 멀티턴 dialogue 전체로 existing memory를 retrieval
   - retrieved memory와 dialogue를 기반으로 fact extraction 수행
   - 추출 결과가 비어 있으면 `NO_ADD`
   - 하나 이상 추출되면 `ADD`

3. **P3: Fact Extraction → Fact Retrieval → Binary**
   - 멀티턴 dialogue에서 먼저 fact extraction
   - 추출된 각 fact를 query로 existing memory retrieval
   - fact별로 LLM binary routing 수행
   - 하나라도 `ADD`이면 전체 dialogue를 `ADD`로 판단

---

## 2. 평가 데이터

총 **100개**의 balanced test set을 사용하였다.

| 유형 | 개수 | Gold |
|---|---:|---|
| AlpsBench memory-add dialogue | 50 | ADD |
| General QA NO_ADD | 25 | NO_ADD |
| Existing-memory repeat NO_ADD | 25 | NO_ADD |
| **Total** | **100** | **ADD 50 / NO_ADD 50** |

평가 지표:

- Accuracy
- Precision
- Recall
- F1
- False Add Rate
- TP / TN / FP / FN

---

## 3. 전체 결과

| Pipeline | N | Accuracy | Precision | Recall | F1 | False Add Rate | TP | TN | FP | FN |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| **P1: Dialogue → Retrieval → Binary** | 100 | **0.9600** | **1.0000** | **0.9200** | **0.9583** | **0.0000** | 46 | 50 | 0 | 4 |
| P2: Dialogue → Retrieval → Extract | 100 | 0.9400 | **1.0000** | 0.8800 | 0.9362 | **0.0000** | 44 | 50 | 0 | 6 |
| P3: Extract → Fact Retrieval → Binary | 100 | 0.8700 | **1.0000** | 0.7400 | 0.8506 | **0.0000** | 37 | 50 | 0 | 13 |

---

## 4. Test Type별 Accuracy

| Test Type | N | P1 | P2 | P3 |
|---|---:|---:|---:|---:|
| AlpsBench ADD | 50 | **0.9200** | 0.8800 | 0.7400 |
| General QA NO_ADD | 25 | **1.0000** | **1.0000** | **1.0000** |
| Repeat-existing NO_ADD | 25 | **1.0000** | **1.0000** | **1.0000** |

---

## 5. 주요 결과

### P1이 가장 높은 성능

`Dialogue → Retrieval → Binary`가 전체적으로 가장 높은 성능을 보였다.

```text
Accuracy : 96.0%
Recall   : 92.0%
F1       : 95.83%
```

특히 ADD 50개 중 **46개를 탐지**했고, 놓친 경우는 4개였다.

### 세 방법 모두 False Add는 발생하지 않음

모든 pipeline에서:

```text
FP = 0
False Add Rate = 0%
Precision = 100%
```

이었다.

즉 이번 test set에서는 **저장하면 안 되는 dialogue를 잘못 ADD로 판단한 사례가 없었다.**

General QA와 기존 memory 반복 케이스 모두 100% 정확하게 NO_ADD로 분류하였다.

### 차이는 대부분 ADD Recall에서 발생

성능 차이는 negative 처리보다 **새 memory를 얼마나 놓치지 않는가**에서 발생했다.

| Pipeline | ADD Recall | Missed ADD |
|---|---:|---:|
| **P1** | **92%** | **4** |
| P2 | 88% | 6 |
| P3 | 74% | 13 |

즉 pipeline이 복잡해질수록 오히려 false negative가 증가했다.

---

## 6. Pipeline별 해석

### P1: Dialogue → Retrieval → Binary

가장 높은 성능을 기록하였다.

전체 multi-turn context를 그대로 활용한 뒤 relevant existing memory와 비교하기 때문에, memory-worthy signal이 중간 단계에서 손실되지 않는 것으로 보인다.

```text
Dialogue
   ↓
Memory Retrieval
   ↓
Binary Route
```

현재 결과에서는 가장 단순하면서 가장 높은 Recall과 F1을 기록하였다.

### P2: Dialogue → Retrieval → Fact Extraction

P1 대비 약간 낮은 성능을 보였다.

```text
P1 Recall = 92%
P2 Recall = 88%
```

Fact extraction 단계에서 실제 저장할 정보가 존재하더라도 `[]`가 반환되면 즉시 NO_ADD가 되기 때문에, extraction 자체가 추가적인 false negative source가 된다.

다만 별도의 binary routing call이 필요하지 않아 pipeline이 단순하다는 장점이 있다.

### P3: Fact Extraction → Fact Retrieval → Binary

가장 낮은 Recall을 기록하였다.

```text
Recall = 74%
FN = 13
```

구조상 error propagation이 가장 크다.

```text
Dialogue
   ↓
Fact Extraction
   ↓
Fact-level Retrieval
   ↓
Binary Routing
```

Fact extraction에서 정보를 놓치면 이후 retrieval이나 binary routing에서 복구할 수 없다.

또한 fact가 여러 개 추출되면 retrieval과 LLM routing 호출도 증가하기 때문에 latency와 비용 측면에서도 가장 무거운 구조다.

---

## 7. 결론

이번 100개 balanced test에서는 **P1: Dialogue → Retrieval → Binary**가 가장 높은 성능을 보였다.

```text
P1
Accuracy = 96.0%
F1       = 95.83%
Recall   = 92.0%

P2
Accuracy = 94.0%
F1       = 93.62%
Recall   = 88.0%

P3
Accuracy = 87.0%
F1       = 85.06%
Recall   = 74.0%
```

세 방법 모두 `False Add Rate = 0%`였기 때문에, 성능 차이는 대부분 **ADD를 놓치는 false negative**에서 발생하였다.

현재 결과만 보면:

```text
Dialogue → Retrieval → Binary
```

구조가 가장 단순하면서도 가장 높은 성능을 보였고, **fact extraction을 routing 이전에 추가한다고 해서 성능이 개선되지는 않았다.**

특히 P3처럼 `fact extraction → retrieval → binary`로 단계를 늘리면 extraction error가 누적되어 Recall이 크게 감소하는 경향이 관찰되었다.
