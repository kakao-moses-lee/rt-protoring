# Project Spec: Real-time Proctoring Tool (WebRTC)

## 1. 개요
약 20명 규모의 시험 감독을 위한 경량화된 웹 기반 모니터링 도구.
Python(Backend)과 React(Frontend)를 사용하여, 낮은 지연 시간(<1초)의 화면 공유 및 감독 기능을 제공한다. DB 없이 In-Memory로 상태를 관리한다.

## 2. 사용자 역할 (Actors)
1. **감독자 (Supervisor)**: 방을 생성하거나, 타 감독자가 생성한 방에 접속하여 모니터링한다.
2. **수험자 (Student)**: 방에 접속하여 본인의 전체 화면을 공유한다.

## 3. UI/UX 흐름도 및 기능 명세

### Step 0: 랜딩 페이지 (Landing Page)
* **URL**: `/`
* **UI 구성**:
    * **역할 선택 탭**: [감독자] / [수험자] (Default: 수험자)
    * **식별자 입력**: `input[type="text"]` (Placeholder: "이름 또는 학번을 입력하세요")
    * **입장 버튼**: "대기방 입장"
* **로직**:
    * 입력값 없이 입장 불가.
    * 입장 시 세션 스토리지 또는 전역 상태에 `role`과 `username` 저장.
    * `/lobby`로 라우팅.

### Step 1: 대기방 (Lobby)
* **URL**: `/lobby`
* **UI 구성**:
    * **헤더**: 내 정보 표기 (ex: "홍길동(수험자)님 환영합니다")
    * **방 목록 영역**: 현재 개설된 방 카트 리스트 (Grid 형태)
        * **방 카드 정보**: `방 제목`, `참여 인원 (N/6)`
        * **클릭 액션**: 해당 방으로 입장 (`/room/{roomId}`)
    * **(감독자 전용) 방 만들기 버튼**: 우측 하단 또는 상단에 배치.
        * 클릭 시 모달 Open -> `방 제목` 입력 -> 확인 -> 방 생성 및 자동 입장.
* **기술 요구사항**:
    * WebSocket을 통해 실시간으로 방 생성/삭제/인원수 변경 정보를 갱신받아야 함.

### Step 2: 감독 방 (Room)
* **URL**: `/room/{roomId}`
* **공통 UI**:
    * **Grid Layout**: 2x3 (총 6칸) 고정 그리드.
    * **나가기 버튼**: 로비로 이동.

#### A. 수험자 (Student) View
* **자신의 슬롯**:
    * 초기 상태: 본인의 `username` 표시 + [화면 공유 시작] 버튼.
    * **화면 공유 시작** 클릭 시:
        * 브라우저 네이티브 화면 공유 권한 요청.
        * **성공 시**: "공유 중입니다" 텍스트로 변경 + **화면 테두리에 5px 초록색(#00FF00) 보더** 표시 (피드백 장치).
        * **거절 시**: 아무 일도 일어나지 않음.
* **타인 슬롯**: 비활성화 (타인의 화면은 볼 필요 없음, 혹은 블라인드 처리).

#### B. 감독자 (Supervisor) View
* **권한 구분**:
    * **Host (방장)**: 방을 생성한 사람. [방 종료] 버튼 노출.
    * **Guest (참관)**: 그냥 접속한 사람. [방 종료] 버튼 미노출.
* **모니터링 Grid**:
    * 수험자가 입장하면 해당 슬롯에 `username` 표시.
    * 수험자가 화면 공유를 시작하면 실시간 Video Stream 표시.
    * **Hover Interaction**:
        * Video 슬롯에 마우스 오버(Hover) 시, CSS `transform`을 이용하여 해당 화면이 **가장 위쪽 레이어(z-index)에서 1.5배~2배 확대**되어 프리뷰 제공.
        * 마우스 아웃 시 원상복구.
* **방 종료 로직 (Host Only)**:
    * 종료 클릭 시 해당 방의 모든 소켓 연결 종료 -> 모든 유저 강제 로비 이동 -> 방 목록에서 삭제.

## 4. 기술 스택 및 제약 사항 (Technical Constraints)

### Backend
* **Language**: Python 3.10+
* **Package Manager**: uv
* **Framework**: FastAPI (Web Server), python-socketio (Signaling)
* **Media Server**: `aiortc` (WebRTC SFU/MCU 역할 대체)
    * **중요**: 수험자의 Upstream은 1개만 받고, 다수의 감독자에게 복제(Clone)하여 전송.
* **Database**: None (Python `dict` 전역 변수로 메모리 관리)

### Frontend
* **Framework**: React + Vite
* **Style**: CSS Modules or Tailwind (개발 편의성)
* **Browser Support**: Chrome (Chromium based) 최우선.

### Performance & Security
* **Latency**: 목표 500ms 이내 (Local Network/Good Wifi 기준).
* **Capacity**:
    * 방 정원: 수험자 6명 (Hard Limit).
    * 감독자 접속: 인원수 제한에 포함되지 않음 (N명 가능).
* **Security**:
    * 수험자는 감독자의 화면을 볼 수 없음 (Media Track 방향성 제어).
    * URL 역추적 방지: 별도 조치 없이 로비 진입 모델로 해결 (모두가 로비를 거침).

## 5. 데이터 모델 (In-Memory Example)

```python
# Rooms Structure
rooms = {
    "room_uuid": {
        "title": "선형대수학 중간고사",
        "host_id": "supervisor_session_id",
        "students": {
            "student_session_id": {
                "identity": "20230001 김학생",
                "is_sharing": True,
                "pc": RTCPeerConnectionObject
            }
        },
        "supervisors": ["sup_session_id_1", "sup_session_id_2"]
    }
}