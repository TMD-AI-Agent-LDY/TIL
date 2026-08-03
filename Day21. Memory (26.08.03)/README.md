# Day21. Memory (26.08.03)

## LangGraph Memory

- LangGraph에서 대화 상태를 기억하고 복원하는 방법을 학습한다.
- Short-term Memory와 Long-term Memory의 역할 및 저장 범위를 구분한다.
- Checkpointer, State History, Time Travel, Interrupt를 이용해 실행 상태를 추적하고 수정한다.
- Store와 Tool을 활용해 사용자 정보를 장기적으로 저장하고 Agent가 스스로 기억을 관리하도록 구성한다.

![강의 표지](images/001.png)

## 학습 목차

- Short-term Memory
  - Short-term Memory 복습
  - Checkpointer
  - State History
  - Time Travel
- Long-term Memory
  - Long-term Memory 복습
  - Store
  - Tool 기반 메모리 관리
- 참고자료
  - LangGraph 설계 철학

![학습 목차](images/002.png)

## 1. Short-term Memory

### 1.1 메모리가 필요한 이유

- 다중 턴 대화에서 이전 대화의 맥락을 고려해 Tool을 호출하려면 상태를 저장해야 한다.
- LangGraph의 메모리는 저장 범위에 따라 두 종류로 나뉜다.
  - `Short-term memory`: Checkpointer를 통해 하나의 대화 스레드 상태를 저장한다.
  - `Long-term memory`: Store를 통해 스레드와 세션을 넘어 유지할 정보를 저장한다.

![메모리 종류](images/003.png)

### 1.2 Checkpointer로 대화 상태 저장

- Checkpointer는 스레드별 대화 상태를 저장하는 저장소다.
- 실습에서는 메모리 기반 저장소인 `InMemorySaver`를 사용한다.
- `create_agent()`의 `checkpointer` 인자에 저장소를 전달하면 Agent가 대화 상태를 기록할 수 있다.

![Agent에 Checkpointer 연결](images/004.png)

- 호출 시 `configurable.thread_id`를 지정하면 대화가 해당 스레드에 저장된다.
- 같은 `thread_id`를 사용한 후속 호출은 이전 대화 맥락을 이어받는다.
- 서로 다른 `thread_id`는 독립된 대화 세션처럼 동작한다.

![thread_id로 대화 저장](images/005.png)

### 1.3 LangChain Agent와 LangGraph의 Checkpointer

- LangChain Agent에서는 `create_agent()`를 생성할 때 `checkpointer`를 전달한다.

![LangChain Agent의 Checkpointer](images/006.png)

- 직접 만든 LangGraph에서는 `workflow.compile(checkpointer=memory)` 형태로 그래프 컴파일 시 저장소를 전달한다.

![LangGraph 컴파일 시 Checkpointer 전달](images/007.png)

- Agent 호출 방식은 동일하게 메시지와 `thread_id`가 포함된 config를 전달한다.
- LangChain Agent도 내부적으로 LangGraph의 상태 관리 기능을 이용하므로 같은 방식으로 스레드 기억을 사용할 수 있다.

![Agent 호출과 대화 저장](images/008.png)

## 2. State History

### 2.1 State, Node, Edge

- `State`: 그래프 실행 중 공유하고 갱신하는 데이터 공간이다.
- `Node`: State를 읽고 새로운 결과를 작성하는 작업 단위다.
- `Edge`: Node 사이의 실행 순서를 연결한다.
- `Conditional Edge`: State 또는 실행 결과에 따라 다음 Node를 선택한다.

![State와 Node 구조](images/009.png)

### 2.2 현재 상태 조회 - get_state()

- `get_state(config)`는 특정 스레드의 현재 상태를 `StateSnapshot`으로 반환한다.
- 주요 속성은 다음과 같다.
  - `values`: 현재 State에 저장된 값
  - `next`: 다음에 실행할 Node
  - `config`: 스레드와 체크포인트를 식별하는 정보
  - `metadata`: 저장 출처와 실행 단계
  - `parent_config`: 이전 체크포인트와의 연결 정보

![get_state와 StateSnapshot](images/010.png)

### 2.3 전체 이력 조회 - get_state_history()

- `get_state_history(config)`는 특정 스레드의 모든 체크포인트를 조회한다.
- 결과는 여러 개의 `StateSnapshot`이며, 가장 최근 상태부터 과거 상태 순으로 반환된다.
- 각 스냅샷을 확인하면 메시지가 추가되고 Node가 실행된 과정을 단계별로 추적할 수 있다.

![get_state_history 동작](images/011.png)

## 3. Time Travel과 Interrupt

### 3.1 Time Travel 개념

- Time Travel은 저장된 과거 체크포인트를 기준으로 새로운 실행 분기를 만드는 기능이다.
- 주요 활용 방식은 다음과 같다.
  - 과거 상태로 돌아가 다른 입력으로 다시 실행하기
  - 잘못된 메시지나 Tool 호출을 수정한 뒤 이어서 실행하기
  - 실행 도중 멈추고 사람이 검토한 뒤 재개하기

![Time Travel과 Interrupt](images/012.png)

### 3.2 Time Travel용 그래프 구성

- Tool 호출이 가능한 모델과 사용할 Tool을 먼저 정의한다.

![Model과 Tool 정의](images/013.png)

- Agent Node는 현재 메시지를 모델에 전달하고 모델의 응답을 State에 추가한다.
- Tool을 바인딩한 모델을 사용하면 모델이 필요한 Tool과 인자를 선택할 수 있다.

![Agent Node 생성](images/014.png)

- 기존 방식에서는 모델이 요청한 Tool을 찾아 직접 실행하고, 결과를 `ToolMessage`로 변환해 State에 추가했다.
- 이때 원래 Tool 호출과 결과를 연결하기 위한 `tool_call_id`가 필요하다.

![기존 Tool Node 구현](images/015.png)

- `ToolNode`를 사용하면 Tool 탐색, 실행, 결과 메시지 변환을 미리 구현된 Node로 대체할 수 있다.
- 직접 구현해야 할 코드가 줄고 Tool 실행 방식이 일관돼진다.

![ToolNode 활용](images/016.png)

- Conditional Edge는 마지막 메시지의 `tool_calls` 존재 여부에 따라 Tool Node 또는 종료 지점으로 라우팅한다.

![Conditional Edge와 그래프 컴파일](images/017.png)

### 3.3 과거 상태에서 Fork 생성

- `get_state_history()`로 체크포인트 목록을 가져온 뒤 수정할 시점을 선택한다.

![과거 이력 조회](images/018.png)

- `update_state()`는 선택한 체크포인트를 기준으로 새로운 상태 분기를 만든다.
- 주요 인자는 다음과 같다.
  - `config`: 수정 기준이 되는 체크포인트
  - `values`: 새로 반영할 State 값
  - `as_node`: 해당 업데이트를 수행한 것으로 간주할 Node

![update_state로 Fork 생성](images/019.png)

### 3.4 Interrupt로 실행 제어

- 그래프를 컴파일할 때 Checkpointer와 함께 실행 중단 지점을 지정할 수 있다.
  - `interrupt_before`: 지정한 Node 실행 전 중단
  - `interrupt_after`: 지정한 Node 실행 후 중단
- Tool 실행 전 사용자 승인이나 실행 후 결과 검수가 필요한 Human-in-the-loop 흐름에 활용한다.

![Interrupt 설정](images/020.png)

- 그래프를 실행하면 지정한 중단 지점에서 State가 저장되고 다음 Node 실행을 기다린다.

![Interrupt가 적용된 그래프 실행](images/021.png)

- `get_state(config)`로 중단된 현재 상태와 다음 실행 Node를 확인할 수 있다.

![중단 상태 조회](images/022.png)

- `StateSnapshot.next`를 통해 실제로 어느 Node 앞에서 멈췄는지 확인한다.
- `config`와 `metadata`를 함께 확인하면 스레드, 체크포인트, 실행 단계까지 추적할 수 있다.

![StateSnapshot 상세 구조](images/023.png)

### 3.5 상태 수정 후 재실행

- 현재 State에서 Tool 호출을 요청한 `AIMessage`를 찾는다.

![Tool 호출 AIMessage 조회](images/024.png)

- 필요한 값을 수정한 뒤 `update_state()`로 State를 갱신한다.
- 새로운 사용자 입력을 추가하지 않고 저장된 상태에서 계속 실행할 때는 `invoke(None, config)`를 사용한다.

![상태 수정 및 재실행](images/025.png)

- 검토 과정을 함수화하면 현재 스냅샷 조회, 이상 행동 검사, 승인 또는 수정, 실행 재개 과정을 자동화할 수 있다.

![검토 흐름 자동화](images/026.png)

## 4. Runtime과 State

### 4.1 Agent 실행 정보의 구분

- `Runtime`은 Agent가 실행되는 동안 참조하는 공유 정보다.
  - `Context`: 사용자 ID, 데이터베이스 연결 정보처럼 실행 중 변하지 않는 값
  - `Store`: 사용자 선호나 프로필 같은 장기 기억
  - `Stream writer`: 내부 동작이나 이벤트를 실시간으로 전달하는 기능
- `State`는 Agent 실행 과정에서 만들어지고 변경되는 결과물이다.
  - 메시지 기록
  - Tool 실행 결과
  - Node가 생성한 중간 데이터

![Runtime과 State](images/027.png)

## 5. Long-term Memory

### 5.1 장기 기억의 역할

- 단기 기억은 하나의 대화 스레드 안에서만 유지된다.
- 장기 기억은 스레드와 세션을 넘어 사용자 정보와 선호를 유지한다.
- Store의 데이터는 다음 두 요소로 식별한다.
  - `namespace`: 폴더처럼 관련 기억을 묶는 영역
  - `key`: namespace 안에서 개별 기억을 구분하는 이름

![Long-term Memory와 Store](images/028.png)

### 5.2 Store 생성

- 실습에서는 `InMemoryStore`를 사용한다.
- 실제 서비스에서는 영속성을 제공하는 데이터베이스 기반 Store로 대체할 수 있다.

![InMemoryStore 생성](images/029.png)

### 5.3 namespace 정의

- namespace는 사용자와 애플리케이션 맥락을 조합한 튜플로 구성할 수 있다.
- 사용자별로 고유한 namespace를 사용하면 서로의 기억이 섞이는 것을 방지할 수 있다.

![namespace 정의](images/030.png)

### 5.4 기억 저장 - put()

- `store.put(namespace, key, value)`로 장기 기억을 저장한다.
- `value`에는 사용자 선호, 사실, 언어 설정 등 Agent가 다음 대화에서 활용할 정보를 구조화해 기록한다.

![Store에 장기 기억 저장](images/031.png)

### 5.5 기억 조회 - get()과 search()

- `get(namespace, key)`는 지정한 하나의 기억을 조회한다.

![get으로 단일 기억 조회](images/032.png)

- `search(namespace)`는 namespace에 포함된 기억 목록을 검색한다.

![search로 기억 검색](images/033.png)

### 5.6 Agent 응답에 기억 주입

- Store에 저장하는 것만으로 모델이 자동으로 기억을 참고하지는 않는다.
- 모델 호출 전에 사용자의 namespace를 결정하고 Store에서 관련 정보를 불러와 프롬프트에 주입해야 한다.
- 이를 위해 사용자 식별 정보를 담는 Context와 기억 주입용 Middleware를 구성한다.

![Store 기억을 모델에 주입하는 구조](images/034.png)

- Middleware는 모델 호출 직전에 Runtime의 Context와 Store를 조회해 시스템 메시지에 사용자 기억을 추가한다.
- 기억 조회와 프롬프트 조합을 공통 계층에 모아 여러 Agent 호출에 재사용할 수 있다.

![기억 주입 Middleware](images/035.png)

## 6. LangGraph에서 Store 사용

### 6.1 그래프에 Store 연결

- LangGraph에서는 그래프를 컴파일할 때 `store` 인자로 장기 기억 저장소를 전달한다.
- Checkpointer는 스레드 상태를, Store는 스레드 밖에서도 유지할 정보를 담당한다.

![그래프 컴파일 시 Store 연결](images/036.png)

### 6.2 Node에서 BaseStore 사용

- Node 함수는 State와 Store를 입력으로 받아 장기 기억을 읽거나 쓸 수 있다.
- `BaseStore`는 LangGraph가 정의한 저장소 표준 인터페이스다.
- Node가 구체적인 저장 기술보다 공통 인터페이스에 의존하면 저장소 교체와 테스트가 쉬워진다.

![Node와 BaseStore](images/037.png)

### 6.3 Agent Node에서 기억 쓰기

- 사용자 ID로 namespace를 구성한다.
- 대화에서 추출한 사용자 정보를 Store에 저장한다.
- 저장이 끝난 뒤 모델이 생성한 메시지를 State에 반환한다.

![장기 기억 쓰기](images/038.png)

### 6.4 Agent Node에서 기억 읽기

- Agent 실행 전 Store에서 사용자의 기억을 검색한다.
- 검색 결과를 프롬프트에 포함해 사용자 맥락에 맞는 응답을 생성한다.

![장기 기억 읽기](images/039.png)

### 6.5 기억 관리 Tool 구성

- 사용자 정보 조회와 저장 기능을 Tool로 분리하면 Agent가 필요할 때 기억을 읽고 갱신할 수 있다.

![사용자 정보 Tool 정의](images/040.png)

- `create_agent()`에 Tool, Store, Context schema를 전달해 장기 기억을 사용하는 Agent를 생성한다.

![Store를 사용하는 Agent 생성](images/041.png)

## 7. Tool 기반 메모리 관리

### 7.1 기억 관리의 주체를 Agent로 전환

- 서비스 운영자가 모든 대화를 직접 확인해 Store에 저장하는 방식은 확장하기 어렵다.
- 기억할 정보를 판단하고 저장하는 행위를 Agent의 추론 과정에 포함하면 메모리 관리가 자동화된다.
- 단, 민감정보 저장 기준과 사용자 동의, 삭제 정책은 애플리케이션 수준에서 별도로 통제해야 한다.

![Agent 중심 기억 관리](images/042.png)

### 7.2 기억 저장 Tool 정의

- 기억 저장 Tool은 사용자 식별 정보와 저장할 내용을 받아 Store를 갱신한다.
- Runtime을 통해 현재 사용자 Context와 Store에 접근한다.

![기억 관리 Tool 정의](images/043.png)

### 7.3 Tool Node 정의

- Tool Node는 모델의 Tool 호출 요청을 실행하고 결과를 메시지로 반환한다.
- 기억 저장 Tool도 일반 Tool과 동일한 Tool 호출 흐름에 포함된다.

![기억 관리 Tool Node](images/044.png)

### 7.4 조건부 그래프 구성

- Agent Node가 기억 저장을 요청했는지 검사한다.
- 저장 요청이 있으면 `save_node`로, 없으면 종료 지점으로 이동하도록 Conditional Edge를 구성한다.

![Tool 기반 메모리 그래프](images/045.png)

## 8. LangGraph 설계 철학

### 8.1 Runtime과 Store를 명시적으로 전달하는 이유

- LangGraph는 Node가 필요로 하는 실행 환경과 저장소를 함수 입력으로 명시한다.
- 숨겨진 전역 상태보다 의존 관계가 분명해져 실행 흐름을 파악하기 쉽다.

![LangGraph 설계 철학](images/046.png)

- 명시적인 의존성 전달은 코드가 다소 길어 보일 수 있지만, 더 견고한 엔지니어링 구조로 이어진다.

![명시적인 구조의 목적](images/047.png)

- Node 함수의 입력 파라미터에 State, Runtime, Store가 드러나면 다음 이점이 있다.
  - 테스트에서 저장소와 Context를 쉽게 교체할 수 있다.
  - Node가 어떤 데이터에 의존하는지 코드만으로 확인할 수 있다.
  - 기능 변경의 영향 범위를 추적하기 쉬워 유지보수성이 높아진다.

![LangChain과 LangGraph 구조 비교](images/048.png)

## 핵심 정리

- Checkpointer는 `thread_id`를 기준으로 하나의 대화 스레드 상태를 저장한다.
- `get_state()`는 현재 스냅샷을, `get_state_history()`는 전체 체크포인트 이력을 조회한다.
- Time Travel은 과거 체크포인트에서 State를 수정해 새로운 실행 분기를 만든다.
- Interrupt는 Tool 실행 전후에 그래프를 멈춰 사람의 승인과 검수를 연결한다.
- Store는 스레드와 세션을 넘어 유지할 장기 기억을 `namespace`와 `key`로 관리한다.
- Agent가 Store를 실제 응답에 활용하려면 기억을 모델 프롬프트에 주입하거나 기억 관리 Tool을 제공해야 한다.
- LangGraph는 State, Runtime, Store 의존성을 명시해 테스트 가능성과 유지보수성을 높인다.

## 참고자료

- [LangGraph - Add memory](https://docs.langchain.com/oss/python/langgraph/add-memory)
- [LangChain - Long-term memory](https://docs.langchain.com/oss/python/langchain/long-term-memory)
- [LangGraph - Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [LangGraph Graph API Reference](https://reference.langchain.com/python/langgraph/graphs)
- [LangGraph Store API Reference](https://reference.langchain.com/python/langgraph/store/)
