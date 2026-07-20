# Day13. LangChain (26.07.20)

#### 1. What is LangChain

- Agent
  ![001](images/001.png)
  - 목표를 달성하기 위해 스스로 판단하고 도구를 사용해 행동하는 시스템
    - 기존 : 개발자가 직접 코드 작성
    - 코딩 어시스턴트 : 개발자가 AI의 도움을 받아 코드 작성 및 수정
      - 개발자 : 이 기능 어떻게 구현해?
      - AI : 이 기능은 이렇게 코드 작성하면 됩니다.,
      - 개발자 : 코드 검수 후 구현
    - 코딩 에이전트 : AI가 코드 작성 및 수정
      - 개발자 (사람) : 이 기능 구현해 줘.
      - AI : 구현 완료됐습니다.
- 랭체인이란?
  - LLM 기반 애플리케이션과 Agent를 빠르고 쉽게 개발할 수 있는 프레임워크
  - Agent Architecture
    ![002](images/002.png)
    - 사용자의 입력(질문, 지시)에 대해 적절한 행동을 취하도록 추론한 뒤 행동하는 주체
    - 구성 요소
      - Model (LLM)
      - Tools (함수)
- 랭체인 시작하기
  - pip install -U langchain
    ![003](images/003.png)
  - API 키 os 환경 변수 저장 (.env 활용)
    - 모델 호출 시 API 키를 매번 입력하지 않아 코드가 간결해짐.

#### 2. Model

- Model I/O
  - 1. 텍스트 생성 (Text Generation)
    - 사람처럼 텍스트 생성
  - 2. 구조화된 답변 생성 (Structured Output)
    - JSON, 리스트, XML
  - 3. 멀티 모달 (Multimodality)
    - 이미지, 영상, 음성 생성
  - 4. 추론
  - 5. Tool Calling
       ![004](images/004.png)
  - init_chat_model 모듈을 가져와 모델 설정
    ![005](images/005.png)
  - invoke() 메서드를 사용해 답변(텍스트) 생성
  - 모델 답변 (AIMessage 객체) 구조
    - usage_metadata
      - input_tokens : 입력 토큰 (안녕하세요 당신은 누구십니까?)
      - output_tokens : 응답 토큰
      - reasoning : 모델이 최종 텍스트를 생성하기 전, 혼자 속으로 논리를 전개하고 생각하는 데 사용한 토큰
        ![006](images/006.png)
- parameter
  - init_chat_model
    ![007](images/007.png)
    - temperature (창의성)
      - 답변의 무작위성을 조절하여 창의적(High) 혹은 논리적(Low) 성향 결정 (0~1.0)
      - max_tokens (용량)
        - 응답의 최대 길이를 제한하여 비용 관리 및 출력 통제 (토큰)
      - timeout (시간)
        - 응답 대기 시간을 제한하여 시스템의 무한 대기(Hang) 방지 (초)
      - max_retries (안정성)
        - 네트워크 오류 발생 시 자동 재시도를 통해 연결 성공률 확보
  - OpenAI 특화 파라미터
    - reasoning_effort : 모델이 최종 텍스트를 생성하기 전, 혼자 속으로 논리를 전개하고 생각하는 데 얼마나 노력할지 설정
      - none : 추론력 비활성화
      - low : 약간의 추가 사고
      - medium : 계획 수립 약간 복잡한 추론
      - high : 코딩같이 매우 복잡한 추론
- 주요 메서드
  - stream( ) : chunk(말뭉치) 단위로 잘라서 하나씩 답변 생성 → UX 증가
    ![008](images/008.png)
  - batch( ) : 여러 질문을 병렬적으로 요청 가능 → UX 증가 및 모델 호출을 줄여
    비용 감소
    ![009](images/009.png)
- 메모리 구현
  - 메시지 구성 요소
    - System Message : 개발자가 사전에 작성한 입력 (프롬프트)
    - Human Message : 사용자의 요청(request), 입력
    - AI Message : Model의 답변 (result)
      ![010](images/010.png)
  - 예시 코드 1 – Message 객체 사용
    ![011](images/011.png)
  - 예시 코드 2 – 딕셔너리 포맷 (OpenAI)
    ![012](images/012.png)
    - user - human / ai - assistant
- 구조화된 답변
  - Structured Outputs
  - 다음 작업을 위해 답변을 특정 형태로 파싱할 때 사용
    ![013](images/013.png)
    ![014](images/014.png)
    ![015](images/015.png)

#### 3. Agents

- Tool 기초
  - 날씨 관련 요청이 들어오면 model이 날씨를 검색하는 tool을 호출하고, tool의 검색 결과를 참고해 model이 답변
  - Tool은 함수다
    ![016](images/016.png)
    ![017](images/017.png)
    ![018](images/018.png)
    ![019](images/019.png)
    ![020](images/020.png)
- Tool 응용
  ![021](images/021.png)
- LangChain 통합 Tools
  - 랭체인에 이미 많은 Tool들이 통합 되어있다!
  - [https://docs.langchain.com/oss/python/integrations/tools](https://docs.langchain.com/oss/python/integrations/tools)
- Agent 메모리
  ![022](images/022.png)
  ![023](images/023.png)
  ![024](images/024.png)
  ![025](images/025.png)
  ![026](images/026.png)
- 구조화된 답변 생성
  ![027](images/027.png)
  ![028](images/028.png)
  ![029](images/029.png)
  ![030](images/030.png)

#### 4. LangChain x Streamlit

- PJT
  - PJT 폴더 참고.

#### 5. 참고자료

- System Prompt
- ReAct & Lost in the Middle
  - ReAct
    - ReAct = Reasoning + Acting
    - Plan and Execute
      - 전체 계획 수립 → 단계별 실행 → 결과 정리
      - 복잡한 작업을 작은 하위 작업으로 먼저 나눠야 할 때 적합
        ![031](images/031.png)
    - Reasoning Without Observation
      - 작업 수행 → 평가 → Self-Reflection → 개선된 전략으로 재시도
      - 에이전트가 실패 결과를 보고 스스로 반성한 뒤 다시 시도하는 구조
        ![032](images/032.png)
    - Loop Engineering
      - Core Agent Algorithm : LLM이 업무를 완수할 동안 계속 루프를 돌게 설계하자!
        ![033](images/033.png)
  - Lost in the Middle
    ![034](images/034.png)
- LangGraph & LangSmith
  - 랭그래프
    - 오케스트레이션(Orchestration) 기반 에이전트 → 복잡한 의사결정에 특화
    - 오케스트레이터(지휘자)가 작업을 분해, 위임, 종합
  - 랭스미스
    - 랭체인, 랭그래프로 만든 애플리케이션 추적, 평가, 모니터링
    - 추적
      - LLM 내부의 실행 과정을 시각화
    - 평가
      - 테스트 데이터를 실행시켜 평가 가능
    - 모니터링
      - 실제 서비스 배포 후 상태 확인 (Latency, API 비용, 에러)
  - 랭스미스 사용법
    - API Key 발급
      - Description 작성
      - PAT 선택
      - Create API Key 클릭
        ![035](images/035.png)
    - os 환경변수 설정
      - langchain 라이브러리 초기화
      - langchain, langchain-openai 패키지 재설치
      - os 환경 변수 입력
        ![036](images/036.png)
    - 랭체인 실행 → 랭스미스 확인
      ![037](images/037.png)
    - 결과 확인
      ![038](images/038.png)
