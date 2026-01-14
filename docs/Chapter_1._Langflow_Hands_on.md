# Chapter 1. Langflow Hands on

## 1. 개발 환경 구성 (Windows + WSL2 + Ubuntu 24.04)

오늘 실습은 **Windows 10/11**에서 **WSL2**를 활용해 **Ubuntu 24.04** 환경을 구성하고, 그 위에서 **uv**로 **Langflow OSS(1.6.8)**를 실행하는 방식입니다.

Windows에 설치된 Python/Anaconda 환경과 독립적으로 작동하기 때문에 깔끔하게 시작할 수 있습니다.

### 1-1 WSL2 설치하기

1. 윈도우에서 wsl 시작하기

   ![그림: WSL 시작](이미지 설명)

   - **win 10도 같은 흐름**

   만약 아래 메시지 발생 시 아무 키나 눌러서 하위시스템 설치 후 ➔ 재부팅 진행해주세요.

   ```shell
   wsl.exe --install
   ```

2. **PowerShell**을 관리자 권한으로 실행해주세요.

   ![그림: PowerShell 실행](이미지 설명)

3. 아래 명령어를 입력해 WSL2와 Ubuntu 24.04를 설치합니다.

   ```shell
   wsl --set-default-version 2
   wsl --install -d Ubuntu-24.04
   ```

   설치가 완료되면 사용자명과 비밀번호를 설정하라는 메시지가 나옵니다.

   ```shell
   wsl -d Ubuntu-24.04
   ```

   계정정보(예시): sdclass  
   비밀번호: 1234

   > **wsl --install** 이후 최초 부팅 시 설정한 Ubuntu 계정/비밀번호를 반드시 기록해 두세요.

4. 필요 시 재부팅을 진행합니다.

### 1-2 Ubuntu 초기화 및 필수 도구 설치

1. 시작 메뉴에서 **WSL** 또는 **Ubuntu-24.04**를 검색해 실행합니다.
2. (선택) 패키지 다운로드 속도 개선 - **Kakao Mirror** 설정

   ```bash
   sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.backup
   sudo sed -i 's|http://archive.ubuntu.com|http://mirror.kakao.com|g' /etc/apt/sources.list.d/ubuntu.sources
   ```

   > 이 단계는 선택사항입니다. 건너뛰어도 실습 진행에는 문제 없지만, 설치 시간이 다소 길어질 수 있습니다.

3. 터미널에서 아래 명령어로 최신 패키지와 개발 도구를 설치합니다.

   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo apt install build-essential curl wget git -y
   ```

4. Python 패키지 관리 도구인 **uv**를 설치한 후, 셸을 재시작합니다.

   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   exec bash
   uv --version
   ```

   ```bash
   source $HOME/.local/bin/env
   ```

5. Langflow가 필요로 하는 Python 3.12를 미리 설치해 둡니다.

   ```bash
   uv python install 3.12
   ```

### 1-3 VS Code와 Git 연동

1. Windows에 **VS Code**를 설치한 후, **WSL 확장(Remote - WSL)**을 추가합니다.

   [Download Visual Studio Code - Mac, Linux, Windows](링크)

2. VS Code 왼쪽 하단의 ![그림: WSL 확장](이미지 설명) 아이콘을 클릭한 뒤 **WSL: Ubuntu-24.04에 새 창 열기**를 선택합니다.

3. 터미널을 열고(단축키: Ctrl+\`) 실습 저장소를 클론합니다.

   ```bash
   mkdir -p ~/work && cd ~/work
   git clone https://github.com/Yo-sure/llm-agent-workshop.git
   cd llm-agent-workshop
   ```

4. 폴더에서 열어줍니다.

5. VS Code 확장 탭에서 **Python, Jupyter, Black Formatter**를 찾아 **Install on WSL** 버튼으로 설치합니다.

### 1-4 Langflow OSS 실행하기

1. 프로젝트 루트에서 가상환경을 생성하고 의존성을 설치합니다.

   ```bash
   uv venv .venv
   uv sync
   ```

   의존성 설치에는 몇 분이 소요될 수 있습니다.

2. 설치된 도구로 로컬 python 가상환경을 에디터에 묶어줍니다.

   ```plaintext
   ctrl + shift + p : python select interpreter 검색
   ```

3. Langflow 버전을 확인한 후 실행합니다.

   ```bash
   uv run langflow --version
   uv run langflow run
   ```

4. 터미널에 표시되는 URL(http://127.0.0.1:7860)을 브라우저에서 열면 Langflow UI가 나타납니다.

### 1-5. 브랜치 로드맵

실습은 Git 브랜치를 이동하며 진행됩니다. 각 단계에 맞는 코드와 예제가 준비되어 있습니다.

예시:

| 단계          | 설명                  | 명령어                             |
|---------------|-----------------------|------------------------------------|
| main          | WSL 환경 세팅 및 UI 투어 | `git checkout main`                |
| 01-news-agent | GDELT 뉴스 분석 도구 실습 | `git checkout 01-news-agent`       |

LLM API 키는 실습 중 제공되는 별도 링크(Notion)에서 확인하실 수 있습니다.

[https://bit.ly/251124AGENT](https://bit.ly/251124AGENT)

> **API 키는 .env 파일 또는 Langflow 환경 변수 입력란에만 보관해 주세요. Git에 절대 올리지 마세요.**

다음 단계

환경 설정이 완료되었습니다! 이제 **Chapter 2. Agent Flow & GDELT** 문서로 이동해서 실습을 진행합니다.
