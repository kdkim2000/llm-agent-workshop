# Chapter 0. LLM에서 Agent로, Context Engineering의 중요성

> **Note**
> 
> 학습 목표 이 챕터에서는 아래 내용을 목표로 합니다.
> 
> - LLM의 본질과 한계를 명확히 이해하고 설명할 수 있다
> - Agent가 왜 필요한지, 어떻게 동작하는지 파악할 수 있다
> - DAG 문제와 Context Engineering의 관계를 설명할 수 있다
> - 주요 Agent 설계과정의 역할과 선택 기준을 안다

들어가며: AI의 변곡점에서

GPT-5.1, Claude 4.5 Sonnet, Gemini 2.0 Flash... 최신 LLM은 이미 기본적인 Agent 기능까지 내장하고 있습니다. 하지만 현장에서는 여전히 의문이 생깁니다:

"시각 단순 작업임을 넘어, 복잡한 임무를 자율적으로 처리하려면?"

정답은 **Agent**입니다. 단순히 '답변'만 생성하는 LLM을 넘어, "계획-실행-검증"의 순환을 반복하여 목표를 달성하는 시스템이 필요합니다.

최신 모델들은 이미 Agent 기능을 기본 탑재하고 있지만, 진짜 난제는 따로 있습니다. 바로 복잡한 워크플로우 설계(DAG 문제)와 이를 해결하는 **Context Engineering**입니다.

## 1단계: LLM 기초 이해하기

### 1.1 LLM은 어떻게 작동하는가?

LLM의 핵심은 단순합니다: "다음에 올 단어를 예측하는 확률 모델"

```python
def llm_core(input_tokens, model_weights):
    # 입력 토큰 시퀀스 -> 다음 토큰 확률 분포 계산
    next_token_probabilities = compute_probabilities(input_tokens, model_weights)
    
    # 확률에 따라 다음 토큰 선택
    next_token = sample_from_distribution(next_token_probabilities)
    
    return next_token
```

이 간단한 과정의 반복에서 **창발적 능력(Emergent Abilities)**이 나타납니다:

- 산술 연산
- 논리적 추론 (Chain-of-Thought)
- 코드 생성

텍스트 생성 방법: Transformers를 이용한 언어 생성을 위한 다양한 다코딩 방법

### 1.2 네 가지 시스템적 제약

LLM은 강력하지만, 4가지 근본적 한계가 있습니다:

| 제약              | 설명                                   | 영향                               |
|-------------------|----------------------------------------|------------------------------------|
| Stateless         | 이전 대화의 기억 부족                  | 매 요청마다 컨텍스트 재전달 필요   |
| Context Window    | 처리 가능한 정보량 제한                | 긴 문서/대화 처리 불가             |
| Hallucination     | 허구적인 정보 생성                     | 왜곡/잘못된 RAG 필수              |
| Knowledge Cutoff  | 학습 이후 정보 모름                    | Tool Use로 실시간 정보 보완        |

예시: Stateless 문제

```plaintext
❌ 잘못된 접근
conversation_1 = llm("서울의 8월 평균습도는...")
conversation_2 = llm("비가 오는 날씨인가요?")  # 답변할 수 없음!

✅ 올바른 접근
history = [
    ("user", "서울의 8월 평균습도는..."),
    ("assistant", "서울의 8월 평균습도는 70%입니다.")
]
response = llm(history + [("user", "비가 오는 날씨인가요?")])  # 이제 가능!
```

최신 모델의 Context Window (2025년 11월 기준)

| 모델              | 입력 토큰 한도       | 비고                           | 제공사     |
|-------------------|----------------------|--------------------------------|------------|
| GPT-5.1           | 272,000 (기본) + 128,000 (옵션) | 4.0K 확장                     | OpenAI     |
| Claude 4.5 Sonnet | 200,000 (기본)       | 벡터에서 1M까지 확장 가능      | Anthropic  |
| Gemini 2.0 Flash  | 1,048,576            | 약 1M                          | Google     |

하지만 두 가지 문제는 여전합니다:

1. **절대적 용량 한계**: 대규모 코드베이스는 여전히 단일 요청 불가
2. **'Lost in the Middle' 현상**: 중간 정보는 놓침

### 1.3 핵심 통찰: Context가 전부다

LLM = 시청각의 학생

- 시청각(Context Window)의 정보만 사용 가능
- 다른 시청(Stateless)의 내용은 모름
- 오류 추측(Hallucination)
- 시청 후 삭제(Knowledge Cutoff) 소식 모름

> **더 알아보기: Context Engineering은 무엇인가?**
> 
> Open AI의 Noam Brown은 RAG, ReAct, CoT 같은 현재 기법들을 LLM의 추론 능력 부족을 보완하는 '목발(crutch)'이라고 표현했습니다. 모델이 발전하던 어린 기법일지도 가 풀어줄 것이지만, 현재로서는 필수적입니다.
> 
> - [인터뷰 리뷰에 대한 유튜브](#)

### 체크포인트: 1단계

- [x] LLM의 토큰 예측 메커니즘을 설명할 수 있다
- [x] 4가지 시스템적 제약(Stateless, Context Window, Hallucination, Knowledge Cutoff)을 이해한다
- [x] Context Window 크기가 한계임을 안다
- [x] 간단한 코드로 대화 기록을 관리하는 방법을 설명할 수 있다

## 2단계: Agent 패러다임 이해하기

### 2.1 ReAct: 생각하고 행동하기

2022년 구글의 ReAct(Reason + Act) 논문은 현대 Agent의 기틀을 마련했습니다.

**핵심 루프**: Thought(생각) -> Action(행동) -> Observation(결과)

[그림: ReAct 아키텍처]
- Query -> Thought -> Action -> Observation -> Answer

전통적 LLM vs ReAct 비교

```plaintext
❌ 전통적 LLM
response = llm("서울의 현재 날씨는?") 
# "죄송하지만, 실시간 정보에 접근할 수 없습니다..."

✅ ReAct 패턴: 단계적 접근
# 1. Thought: 문제 분석
thought = llm("서울의 현재 날씨를 알기 위해 '씨 API'를 호출해야겠다.")

# 2. Action: 도구 실행
action = weather_api.get("서울")
observation = execute_tool(action)  # {"온도": 25, "상태": "맑음"}

# 3. Observation: 결과 반영
final_response = llm(f"""
이전 생각: {thought}
관찰 결과: {observation}
새 정보에 따라 생성된 도구 입력 및 호출
""")
# "서울의 현재 날씨는 25도이며, 맑은 상태입니다."
```

### 2.2 Agent의 4가지 핵심 모듈

현대 Agent는 4개 모듈의 상호작용으로 구성됩니다. 이는 다양한 연구와 산업 표준에서 공통적으로 제시하는 구조입니다:

| 모듈       | 역할                          | 예시                   |
|------------|-------------------------------|------------------------|
| Profiling  | Agent 정체성 정의             | "당신은 친절한 AI 비서입니다" |
| Memory     | 장단기 메모리 유지            | 대화 기록, 학습한 정보 저장 |
| Planning   | 목표 달성 계획 수립           | 작업 계획, 도구 선택   |
| Action     | 계획 실행                     | API 호출, 결과 전달    |

[그림: Agent 아키텍처]
- Planning Tool -> Sub Agents -> File System -> System Prompt

> **Info**
> 
> - 삼성SDS 인사이트 – AI 에이전트 핵심 능력 분석
> - Enhans.ai - 자동화 및 에이전트 연구 동향
> 
> 일부 문헌에서는 Profiling 대신 Perception(지각)을 사용하기도 하지만, 핵심 구조는 동일합니다.

메모리 구조:

- 단기 메모리(Short-term): 현재 세션 정보
- 장기 메모리(Long-term): 사용자 선호도, 학습 내용
- 에피소드 메모리(Episodic): 특정 시간-장소와 연관된 개인적 경험 (예: "어제 사용자가 날씨 API 호출했음")
- 시맨틱 메모리(Semantic): 자료-요소와 연결된 일반적 지식-사실 (예: "날씨 API는 OpenWeatherMap 사용")

### 체크포인트: 2단계

- [x] ReAct 패턴의 3단계(Thought -> Action -> Observation)를 설명할 수 있다
- [x] Agent의 4가지 모듈(Profiling, Memory, Planning, Action)을 이해한다
- [x] 단기/장기/에피소드 메모리를 구별할 수 있다
- [x] 간단한 ReAct 패턴 Agent를 구현할 수 있다

실습 예제:

- 날씨 API를 연결한 ReAct 패턴 Agent 만들기
- 대화 기록을 저장하고 참조하는 메모리 시스템 구현

## 3단계: DAG 문제 이해하기

### 3.1 이슈와 현실

Agent 작업은 보통 이렇게 구성됩니다:

```
정보 수집 -> 분석 -> 보고서 작성 -> 결과 전달
```

이는 **DAG(Directed Acyclic Graph, 방향성 비순환 그래프)** 구조입니다.

하지만 현실은 다릅니다:

1. **재시도**: API 호출 실패 시 다시 시도
2. **조건부 분기**: 특정 보안에 따른 도구 사용
3. **순환 경로**: 정보 갱신 후 다시 데이터 검증

> 모두 **순환(Cycle)**이 필요합니다!

### 3.2 실전 사례: 투자 자문 Agent

```python
class InvestmentAdvisoryAgent:
    def analyze_and_recommend(self, user_profile):
        # 1단계: 데이터 수집
        market_data = self.collect_market_data()
        
        # 2단계: 데이터 충분성 -> 불충분하면 추가 데이터 수집하기 필요
        if not self.is_data_sufficient(market_data):
            additional_data = self.collect_additional_data()
            market_data.update(additional_data)
        
        # 3단계: 추천 생성
        recommendations = self.generate_recommendations(market_data, user_profile)
        
        # 4단계: 리스크 평가 -> 높으면 경고 단계로 돌아가기 필요
        risk_score = self.assess_risk(recommendations)
        if risk_score > user_profile.risk_threshold:
            # 문제: DAG는 순환을 제어할 수 없음 -> 다른 방법 필요!
```

### 체크포인트: 3단계

- [x] DAG의 정의와 한계를 설명할 수 있다
- [ ] 재시도, 분기, 순환이 필요함을 안다
- [ ] 실제 워크플로우에서 DAG 문제가 발생하는 시나리오를 설명할 수 있다

## 4단계: Context Engineering 마스터하기

### 4.1 정의

**Context Engineering이란?**

Agent의 작업 흐름 전반에 걸쳐, 목표 달성에 필요한 최적의 정보를, 가장 적절한 시점에, 올바른 형식으로 LLM에게 동적으로 공급하는 모든 기술과 전략

단순 Prompt Engineering + Context Engineering

- **Prompt**: 한 번의 입력 최적화
- **Context**: Agent 전체 흐름의 '작업 기억' 설계

### 4.2 핵심 기술 3가지

1) **RAG (Retrieval-Augmented Generation)**

외부 지식 베이스 참조로 활자 검색과 최신 정보 반영

```python
def rag_enhanced_llm(query):
    # 1. Vector DB에서 관련 문서 검색
    relevant_docs = vector_db.similarity_search(query)
    
    # 2. 검색된 문서로 컨텍스트 생성
    context = f"""
    [관련 정보]
    {relevant_docs}
    [질문]
    {query}
    """
    return llm(context)
```

주요 벡터 DB:

- **ChromaDB**: 간단한 클러스터링
- **Pinecone**: 클라우드 완전 관리형
- **Weaviate**: 오픈소스, 하이브리드 검색

2) **통적 메모리 관리**

대화 기록을 요약하거나 중요 정보만 선별해 컨텍스트에 포함

예시 도구:

- **Mem0**: AI 연령 메모리 레이어

3) **상태 의존적 프롬프트**

Agent 현재 상태에 따라 정보 형식 최적화

| 상태       | 컨텍스트 정보 포함        |
|------------|---------------------------|
| 상태 분석  | 새로운 가능 도구, 새로운 정보 |
| 코드 생성  | 코드 생성, 새로운 정보     |
| 디버깅 중  | 에러 로그, 스택 트레이스, 이전 시도 |

### 4.3 Context Engineering이 DAG 문제를 해결하는 원리

DAG 구조를 직접 수정하는 대신, **컨텍스트를 동적 제어** :

- 상태 분석: "현재 단계 제시도 중" 같은 정보를 컨텍스트에 명시
- 논리적 순환: 컨텍스트에 "문제 해결, 분석 단계로 돌아가 다른 방법 시도" 지침 제공

> 물리적 순환 없이도 논리적 순환 구현!

### 체크포인트: 4단계

- [x] Context Engineering이 Prompt Engineering과의 차이를 설명할 수 있다
- [ ] RAG의 장점과 원리를 이해할 수 있다
- [ ] 동적 메모리 관리의 필요성을 안다
- [ ] 상태에 따른 컨텍스트 프롬프트 작성법을 습득할 수 있다

실습 예제:

- 간단한 RAG 시스템 구축 (Vector DB + LLM)
- 대화 요약 및 맞춤 메커니즘 구현
- 상태별 프롬프트 템플릿 설계

## 5단계: 프레임워크를 선택하기

### 5.1 왜 프레임워크가 필요한가?

Agent는 수많은 외부 도구와 상호작용합니다. 각각 다른 사용환경이라면 복잡도가 기하급수적 증가!

프레임워크 = LLM 연동 + 메모리 관리 + 도구 연결 + 실행 루프

> 개발자는 핵심 비즈니스 로직에만 집중 가능

### 5.2 프레임워크 비교

| 프레임워크  | 핵심 환경                  | 주요 해결 과제                  | 적합한 사용 사례                |
|-------------|----------------------------|--------------------------------|--------------------------------|
| LangFlow    | 시각적 UI, No-Code         | 복잡한 워크플로우, 아이디어 검증 | PoC, 비개발자의 Agent 구축     |
| LangGraph   | 상태 기반, DAG 제어        | DAG 문제 해결                  | 복잡한 상태를 가진 Agent       |
| LangChain   | 대화형, 프롬프트 관리      | 기존 Agent 및 RAG 파이프라인과의 통합 | 효율적인 LLM 애플리케이션      |
| LangFuse    | 데이터 분석, 프레임워크    | 프레임워크 모니터링 및 분석    | 운영 및 유지보수               |

### 5.3 LangGraph가 특별한 이유

**LangGraph = DAG 문제를 위해 특별히 설계된 프레임워크**

핵심 기능:

- 상태 관리: 각 단계별 State 변경 노드로 정의
- 조건부 경로: 상태에 따라 다음 노드 동적 결정
- 순환 제어: 명시적 순환 구조 구현 가능

LangGraph 코드 예시

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

# 1. Agent 상태 정의
class AgentState(TypedDict):
    messages: List[str]
    quality_score: float

workflow = StateGraph(AgentState)

# 2. 노드(단계 함수) 정의
workflow.add_node("analyze", analyze_node)  # 분석 노드
workflow.add_node("review", review_node)    # 검토 노드

# 3. 시작점 설정
workflow.set_entry_point("analyze")

# 4. 조건부 엣지 -> 다음 기능
def decide_next_step(state: AgentState):
    if state["quality_score"] < 0.8:
        return "analyze"  # 품질 미달 -> 다시 분석
    else:
        return END        # 종료 -> 종료

workflow.add_conditional_edges("review", decide_next_step)
workflow.add_edge("analyze", "review")

# 5. 그래프 컴파일
app = workflow.compile()
```

### 5.4 MCP (Model Context Protocol)

**MCP = 모델과 컨텍스트 간 표준화된 통신 프로토콜**

Anthropic이 주도하는 표준으로, 다음을 명확한 태그로 구분:

- 시스템 역할
- 사용자 요청
- 메모리
- 도구 호출
- 코드 스니펫

> 모델이 혼동 없이 일관된 컨텍스트 수신 가능

MCP 활용:

- `/messages`: 대화 기록 엔드포인트
- `/state`: 현재 상태 조회
- `/health`: 서버 상태 확인

### 체크포인트: 5단계

- [ ] 4가지 주요 프레임워크(LangFlow, LangGraph, LangChain, LangFuse)의 차이를 설명할 수 있다
- [ ] LangGraph의 상태 관리와 조건부 엣지 개념을 이해한다
- [ ] 프로젝트 요구사항에 맞는 프레임워크를 선택할 수 있다
- [ ] MCP 프로토콜의 역할을 안다
- [ ] LangGraph 순환 구조를 구현할 수 있다

실습 예제:

- LangChain으로 기본 RAG 파이프라인 구축
- LangGraph로 품질 점수 기반 순환 워크플로우 만들기
- 각 프레임워크의 장단점 분석 후 비교

## 6단계: 프로덕션 준비하기

### 6.1 모델 파라미터 이해하기

| 파라미터           | 설명                       | 권장 값                       |
|--------------------|----------------------------|-------------------------------|
| Temperature        | 창의성 조절 (-2~2)         | 0.0: 결정적, 0.7: 균형, 1.5: 창의적 |
| Top-p              | Nucleus

### 6.1 모델 파라미터 이해하기 (계속)

**Temperature**는 모델의 창의성을 조절하는 중요한 파라미터입니다. 값이 0에 가까울수록 모델은 결정론적으로 작동하며, 가장 높은 확률의 단어를 선택합니다. 반면, 값이 2에 가까워질수록 낮은 확률의 단어도 고려하여 더 창의적이고 예측 불가능한 출력을 생성합니다.

[그림: Temperature에 따른 확률 분포 변화]
- **0**: "A cup of coffee." - 높은 확률의 단어 선택
- **1**: "A cup of courage." - 중간 확률의 단어도 고려
- **2**: "A cup of stars." - 낮은 확률의 단어 선택

**Top-K** 샘플링은 확률 분포에서 상위 K개의 단어만 고려하여 새로운 분포를 생성합니다. 예를 들어, K=2일 경우 상위 2개의 단어만 선택하여 재샘플링합니다.

[그림: Top-K 샘플링]
- 원본 분포: stars, dreams, courage, coffee
- 새로운 분포: courage, coffee

**Top-p** 샘플링은 누적 확률이 p에 도달할 때까지 상위 단어들을 선택합니다. 예를 들어, p=0.9일 경우 확률이 0.9가 될 때까지 단어를 선택합니다.

[그림: Top-p 샘플링]
- 원본 분포: stars, dreams, courage, coffee
- 새로운 분포: courage, coffee

### 6.2 도구 활용 (Tool Use)

Agent가 사용할 수 있는 도구 유형:

| 도구 유형     | 예시                  | 활용                      |
|---------------|-----------------------|---------------------------|
| 웹 검색       | Google, Bing API      | 실시간 정보 수집          |
| 코드 실행     | REPL, Sandbox         | 계산, 데이터 처리         |
| 데이터베이스  | SQL, NoSQL            | 구조화 데이터 조회        |
| API 호출      | REST, GraphQL         | 외부 서비스 연동          |
| 파일 시스템   | Read/Write            | 로컬 파일 관리            |
| 파일 시스템   | Read/Write            | 로컬 파일 관리            |
| 이메일/메시지 | SMTP, Slack API       | 알림 전송                 |

도구 정의 예시 (OpenAI Function Calling):

```json
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "특정 도시의 현재 날씨 정보를 가져옵니다.",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "도시 이름 (예: 서울, 부산)"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "온도 단위"
                    }
                },
                "required": ["city"]
            }
        }
    }
]
```

### 6.3 모니터링 및 평가

LangFuse를 활용한 관련 가능성:

- 토큰 사용량 추적
- 응답 시간 모니터링
- 에러율 분석
- A/B 테스트

## 핵심 요약

우리가 배운 것

| 단계               | 핵심 개념                  | 왜 중요한가                                      |
|--------------------|----------------------------|--------------------------------------------------|
| 1. LLM 기초        | Context 의존성, 4가지 패턴 | LLM의 한계를 이해해야 Agent 필요성을 안다         |
| 2. Agent 패러다임  | ReAct, 도구 사용           | LLM의 "행동" 능력을 부여하는 핵심 구조           |
| 3. DAG 관리        | 순환형 워크플로우          | 순환형 워크플로우는 적절하지 않음                |
| 4. Context Engineering | RAG, 동적 메모리, 상태 인식 | DAG 문제를 컨텍스트 제어로 해결                  |
| 5. 표현력          | LangGraph의 실력 관리      | 표현력 구성을 추상화하고 생산성 향상             |
| 6. 프로덕션        | 모델 패러미터, 도구 활용   | 실질적 배포를 위한 최적화와 모니터링             |

## 참고 자료

필수 읽을거리

- [roadmap.sh/ai-agents](https://roadmap.sh/ai-agents) - AI Agent 로드맵
- [Anthropic's Model Context Protocol](https://example.com) - MCP 공식 문서
- [LangGraph Documentation](https://example.com) - LangGraph 가이드
- [Memo](https://example.com) - AI 메모리 레이어

논문

- **ReAct**: Synergizing Reasoning and Acting in Language Models (2022)
- **Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks** (2020)
- **Chain-of-Thought Prompting Elicits Reasoning in LLMs** (2022)

커뮤니티

- LangChain Discord
- r/LocalLLaMA
- Hugging Face Forums