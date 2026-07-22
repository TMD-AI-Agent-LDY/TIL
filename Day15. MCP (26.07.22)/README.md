# Day15. MCP (26.07.22)

#### 1. MCP 기초

- MCP (Model Context Protocol)
  - Model(LLM)이 다양한 Context(문맥)를 전달받아 활용할 수 있도록 정한 공통 Protocol(규칙)
  - AI 애플리케이션과 외부 Tool·데이터 사이의 연결 방식을 표준화한다.
- 기존 Tool Calling의 한계
  - LangChain의 `@tool`로 함수를 정의하면 이름, 설명, 파라미터가 Tool Schema로 변환된다.
  - Agent를 만들 때 Tool Schema가 Model에 전달되고, Model은 사용자 요청에 맞는 Tool과 인자를 선택한다.
  ![001](images/001.png)
  - Model 제공사마다 Tool Schema 형식이 달라 Tool 개발자가 각 Model에 맞게 구현해야 하는 문제가 있다.
  ![002](images/002.png)
- MCP의 등장
  - 2024년 11월 Anthropic이 공개한 개방형 프로토콜
  - AI Host와 외부 기능을 표준 방식으로 연결해 Tool의 재사용성과 확장성을 높인다.
  - 이후 AI 코드 에디터와 여러 애플리케이션으로 생태계가 확장되었다.
  ![003](images/003.png)
- Playwright MCP 예시
  - Microsoft가 제공하는 브라우저 자동화 MCP Server
  - 클릭, 페이지 닫기, 드래그 앤 드롭, 파일 업로드, Hover, 키 입력, Snapshot 등의 Tool을 제공한다.
  ![004](images/004.png)
- MCP 구성 요소
  - `Host` : LLM이 탑재된 애플리케이션. 예) Codex, Claude Desktop, LLM 기반 Streamlit 앱
  - `Server` : Model에 전달할 Context를 생성하는 Tool의 모음
  - `Client` : Host와 Server 사이의 연결과 통신을 담당하는 다리
  ![005](images/005.png)
- MCP와 API
  - 애플리케이션과 Tool이 요청·응답을 주고받는다는 점은 API와 유사하다.
  - 사용자가 자연어로 요청하면 AI가 필요한 Tool을 추론하고 파라미터와 쿼리를 작성한다.
  - Tool의 실행 결과를 새로운 Context로 받아 최종 답변을 생성한다.
  - MCP 규격을 따르는 Server는 여러 Host에서 유연하게 재사용할 수 있다.
  ![006](images/006.png)

#### 2. MCP Server

- `uv`
  - Rust로 작성된 빠른 Python 패키지·프로젝트 관리 도구
  - 가상 환경 생성(`uv venv`), 패키지 설치(`uv add`, `uv pip install`), 프로젝트 실행(`uv run`)을 하나의 도구로 처리한다.
  ![007](images/007.png)
  - 기존 `venv`와 `pip` 중심의 여러 명령을 더 단순한 흐름으로 바꿀 수 있다.
  ![008](images/008.png)
- MCP Python SDK
  - AI Model이 사용할 Tool을 Python 코드로 개발할 수 있는 SDK
  - `FastMCP`를 이용하면 Tool, Prompt, Resource를 데코레이터 방식으로 선언할 수 있다.
  ![009](images/009.png)
- 기초 Tool 개발 순서
  - 작업 폴더와 가상 환경을 만든다.
  - `uv pip install "mcp[cli]>=1.27,<2"`로 패키지를 설치한다.
  - `FastMCP` 인스턴스를 만들고 함수에 `@mcp.tool()`을 적용한다.
  - 함수의 이름, 타입 힌트, Docstring이 Model이 참고할 Tool 정보가 된다.
  ![010](images/010.png)
  - Entry Point에서 `mcp.run(transport="stdio")`를 호출해 Server를 실행한다.
  ![011](images/011.png)
- Host 설정
  - MCP 설정 파일에 Server 실행 정보를 등록한다.
  - `command` : Server를 실행할 프로그램
  - `args` : 실행 프로그램에 전달할 파일과 인자 목록
  - `cwd` : Server 프로세스가 실행될 작업 디렉터리
  ![012](images/012.png)
- Server 실습
  - Frankfurter 환율 API를 활용해 원·달러 환율을 답변하는 MCP Server를 만든다.
  - API 호출 로직을 Tool로 노출하고 Host 설정에 Server 실행 정보를 추가한다.
  ![013](images/013.png)

#### 3. MCP Host & Client

- Host·Client·Server 연결 구조
  - 하나의 Host는 여러 Client를 통해 여러 MCP Server와 연결될 수 있다.
  - 각 Client는 연결된 Server의 Tool·Prompt·Resource를 Host가 사용할 수 있는 형태로 전달한다.
  ![014](images/014.png)
- MCP 통신의 3개 계층
  - `Client / Server` : Client가 요청하고 Server가 응답하는 역할 계층
  - `Session` : 상태, 응답 대기, 시간 초과 등 한 연결의 상호작용을 관리하는 계층
  - `Transport` : 메시지를 실제로 전달하는 계층
  ![015](images/015.png)
- Transport 방식
  - `stdio` : 표준 입출력으로 로컬 프로세스와 통신한다. 설정과 디버깅이 간단하지만 주로 로컬 환경에 한정된다.
  - `SSE (HTTP)` : 네트워크를 통해 실시간 스트리밍할 수 있다. 원격 환경에 적합하지만 HTTP Server와 보안 설정이 필요하다.
  ![016](images/016.png)
- LangChain MCP Adapter
  - MCP의 저수준 통신 계층을 직접 구현하지 않도록 LangChain이 제공하는 Wrapper
  - MCP Server의 기능을 LangChain Tool로 변환해 Agent에 연결한다.
  ![017](images/017.png)
- Adapter 사용 흐름
  - `langchain-mcp-adapters` 패키지를 설치한다.
  - `MultiServerMCPClient`에 Server의 `command`, `args`, `transport`를 설정한다.
  ![018](images/018.png)
  - `await client.get_tools()`로 Tool 목록을 가져온다.
  - 가져온 Tool로 Agent를 생성하고 `ainvoke()`를 호출한다.
  ![019](images/019.png)
- Host & Client 실습
  - Streamlit 앱에 Playwright MCP Server를 연결해 웹페이지에서 필요한 정보를 찾는다.
  - 직접 만든 환율 MCP Server도 같은 Agent에 연결해 현재 원·달러 환율을 조회한다.
  ![020](images/020.png)

#### 4. Prompt & Resource

- MCP가 제공하는 Context의 종류
  - `Tool` : Model이 판단해 호출하는 실행 가능한 기능
  - `Prompt` : Host 또는 사용자가 선택해 가져다 쓰는 재사용 가능한 프롬프트 템플릿
  - `Resource` : Host 또는 Client가 읽어 Model의 Context로 전달하는 데이터
  ![021](images/021.png)
- Prompt
  - `@mcp.prompt()` 데코레이터로 등록한다.
  - 입력값에 따라 역할, 설명 수준, 출력 형식을 조합하는 프롬프트 템플릿을 만들 수 있다.
  ![022](images/022.png)
- Resource
  - `@mcp.resource("scheme://{parameter}/path")` 형태의 URI 템플릿으로 등록한다.
  - 문서, 사용자 프로필, 설정값처럼 읽기 중심의 Context를 제공할 때 사용한다.
  ![023](images/023.png)

#### 5. Git Branch 전략

- Branch의 역할
  - `main`은 완성되고 검증된 결과물이 모이는 공간이다.
  - 기능이나 담당 영역별 Branch는 독립적으로 작업하는 임시 공간이다.
  - 예) `frontend`, `backend`, `data`
  ![024](images/024.png)
- 기본 작업 흐름
  - `git branch frontend`로 새 Branch를 만든다.
  ![025](images/025.png)
  - `git switch frontend`로 작업 Branch로 이동한다.
  ![026](images/026.png)
  - 파일을 수정한 뒤 `git add`, `git commit`, `git push origin frontend` 순서로 원격 Branch에 반영한다.
  ![027](images/027.png)
- Pull Request와 Merge
  - Pull Request(PR)는 작업 Branch의 변경 사항을 `main`에 합쳐도 되는지 검토를 요청하는 절차다.
  ![028](images/028.png)
  - 검토가 끝나고 충돌이 없으면 Merge해 작업 이력을 `main`에 반영한다.
  ![029](images/029.png)
  - Merge 후 로컬에서 `git switch main`, `git pull origin main`으로 최신 내용을 받는다.
  ![030](images/030.png)
  - 더 이상 필요하지 않은 로컬 Branch는 `git branch -d frontend`로 삭제한다.
  ![031](images/031.png)
- 팀 협업 실습
  - 팀장은 저장소를 만들고 팀원을 초대하며, 팀원은 담당 Branch에서 작업한다.
  - 각자 최신 `main`을 먼저 받은 뒤 수정·Commit·Push·PR·Merge 흐름을 반복한다.
  ![032](images/032.png)
- Conflict
  - 서로 다른 Branch에서 같은 파일의 같은 부분을 다르게 수정하면 충돌이 발생한다.
  - 예제에서는 `frontend`와 `backend`가 `README.md`의 첫 줄을 각각 다르게 수정한다.
  ![033](images/033.png)
  - 먼저 Merge된 변경과 나중 PR의 변경이 겹치면 GitHub가 자동 Merge를 중단한다.
  ![034](images/034.png)
  - `Resolve conflicts`에서 남길 내용을 직접 선택하고 충돌 표시를 제거한다.
  ![035](images/035.png)
  ![036](images/036.png)
  - 해결한 내용을 Commit한 뒤 PR을 Merge한다.
  ![037](images/037.png)
- 충돌을 줄이는 협업 원칙
  - 담당 파일과 폴더를 미리 나눠 같은 영역의 동시 수정을 줄인다.
  - 팀장은 전체 구조와 Merge를 관리하고, 팀원은 할당된 영역을 중심으로 작업한다.
  ![038](images/038.png)
  - 작업 시작 전 최신 `main`을 받고, 큰 변경보다 작고 명확한 Commit과 PR을 만든다.
  - 기능 Branch, Hotfix, Merge의 흐름이 섞여도 `main`은 배포 가능한 상태로 유지한다.
  ![039](images/039.png)
- Branch 정리
  - `main`으로 이동한다.
  - 원격 `main`의 최신 내용을 Pull한다.
  - Merge가 끝난 로컬 Branch와 원격 Branch를 삭제한다.
  ![040](images/040.png)

#### 6. 핵심 정리

- MCP는 AI Host와 외부 Context 제공자를 표준 방식으로 연결한다.
- MCP Server는 Tool뿐 아니라 재사용 가능한 Prompt와 읽기 전용 Resource도 제공할 수 있다.
- Host는 Client를 통해 Server와 연결되고, Session과 Transport 계층이 통신을 담당한다.
- LangChain MCP Adapter를 사용하면 MCP Tool을 LangChain Agent에 쉽게 연결할 수 있다.
- Git 협업에서는 기능별 Branch에서 작업하고 PR 검토 후 `main`에 Merge한다.
- 최신 `main` 동기화, 명확한 담당 영역, 작은 PR이 충돌을 줄이는 핵심이다.

#### 7. 참고자료

- [Model Context Protocol](https://modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Playwright MCP](https://github.com/microsoft/playwright-mcp)
- [LangChain MCP Adapters](https://reference.langchain.com/python/langchain-mcp-adapters)
- [uv Documentation](https://docs.astral.sh/uv/)
- [Frankfurter 환율 API](https://frankfurter.dev/)
