# 프로젝트 구조 개요

- **앱 초기화 (`src/main.py`)
  - `FastAPI()` 인스턴스를 생성하고 CORS 설정을 위해 `CORSMiddleware` 추가.
  - 전역 미들웨어 `authenticate_request(request, call_next)`에서 쿠키의 `session` 값을 읽어 `request.state.user_id`에 주입.
  - `user`, `realstream`, `setting`, `eventlog`, `ai_message`, `TTS`, `analytics`, `anomalyDetect` 라우터를 일괄 등록.
  - `/` 헬스체크 엔드포인트 `health_check_handler()` 제공.

- **미들웨어 및 인증 헬퍼 (`src/core/middleware.py`, `src/dependencies/auth.py`)
  - `auth_middleware(request, call_next)`가 Authorization 헤더의 JWT를 `UserService.decode_jwt()`로 검증해 `request.state.user_id`에 저장.
  - `session_middleware(request, call_next)`가 세션 쿠키를 읽어 사용자 ID를 주입.
  - `generate_secret_key()`, `decode_jwt(token)`, `get_current_user_id(token)` 등 OAuth2 기반 토큰 디코딩/사용자 조회 헬퍼 제공.

- **데이터베이스 계층 (`src/database/connection.py`, `src/database/orm.py`, `src/database/repository.py`)
  - `get_db()`에서 SQLAlchemy 세션을 생성·종료하고 `Base` 선언.
  - ORM 모델: `User.create()/edit()`, `AiMessage`, `Frame`, `Anomaly`, `EventlogSchedule`, `Eventlog` 등 도메인 엔터티 정의.
  - `UserRepository`가 `get_user_by_email()`, `save_user()`, `get_user()`, `edit_user()`, `delete_user()`를 구현.
  - `RealStreamRepository`가 `save_frame_to_db(frame)`과 `save_anomaly_to_db(frame_record_id)`로 영상 프레임/이상행동 기록 저장.
  - `EventlogRepository`가 `create_date_time()`, `create_eventlog()`, `get_eventlog()`, `get_all_eventlogs()` 및 각종 통계 조회 메서드(`get_eventlog_monthly_stats()` 등)를 제공.
  - `AnalyticsRepository`가 월/주/일/시간 단위 통계(`get_monthly_stats()`, `get_weekly_stats()`, `get_daily_stats()`, `get_eventlog_*_stats()`)를 계산.

- **서비스 계층 (`src/service`)
  - `UserService`가 `hash_password()`, `verify_password()`, `create_jwt()`, `decode_jwt()`, `decode_access_token()`으로 비밀번호 해시·JWT 생성/검증·Redis 저장을 수행.
  - `security.py`의 헬퍼가 OAuth2 스킴용 `get_current_user()`, `create_access_token()` 등의 목업 인증 로직을 제공.
  - `utils.py`가 분석 통계를 위한 `get_all_months()`, `get_all_weeks()`, `get_all_days()`, `get_all_hours()`를 계산.

- **스키마 계층 (`src/schema`)
  - 요청 모델: `SignUpRequest`, `LogInRequest`, `EventlogCreate`, `AnalyticsCreate`.
  - 응답 모델: `UserSchema`, `JWTResponse`, `EventlogScheduleResponse`, `EventlogResponse`, `AnalyticsResponse`, `StatsResponse`, `TimeStatsResponse` 등.
  - AI 메시지용 `AiMessageCreate`, `AiMessageUpdate`, `AiMessageSchema`, 설정 화면용 `SettingCreate`, `SettingUpdate`, `UserUpdate` 등 정의.

- **사용자/세션 관련 API (`src/api/user.py`, `src/api/setting.py`)
  - 회원가입 `user_sign_up_handler()`, 로그인 `user_log_in_handler()`에서 `UserService`와 `UserRepository`를 활용.
  - 테스트용 토큰 확인 엔드포인트 `get_token(email)`.
  - 설정 라우터에서 `user_log_out_handler()`로 세션 쿠키 삭제, `get_current_user()`로 JWT·Redis 토큰 검증, `read_settings()`, `delete_account()`, `update_user()`로 사용자 정보 조회/삭제/수정 수행.

- **AI 메시지 & 음성 합성 (`src/api/ai_message.py`, `src/api/TTS.py`)
  - `read_ai_messages()`, `create_ai_message()`, `update_ai_message()`, `delete_ai_message()`로 AI 안내 멘트 CRUD 제공.
  - `save_tts_audio()`가 gTTS를 이용해 MP3 생성, `text_to_speech(message_id)`가 비동기로 저장 작업 큐잉, `get_tts_audio(message_id)`가 생성 파일을 스트리밍.

- **이벤트 로그 및 분석 (`src/api/eventlog.py`, `src/api/analytics.py`)
  - `eventlog_schedule_handler()`, `create_eventlog()`, `read_eventlog()`, `read_all_eventlogs()`로 스케줄 및 로그 조회·생성.
  - `get_monthly_stats()`, `get_weekly_stats()`, `get_daily_stats()`, `get_eventlog_monthly_stats()`, `get_eventlog_weekly_stats()`, `get_eventlog_daily_stats()`로 통계 API 제공.
  - `src/analytics.py.py`는 동일한 라우트를 한국어 라벨 등으로 변형한 중복 버전.

- **실시간 스트림 & 이상행동 처리 (`src/api/realstream.py`, `src/api/anomalyDetect.py`)
  - WebSocket 엔드포인트 `websocket_endpoint()`가 프레임 수신 후 `save_frame_to_db()`/`save_anomaly_to_db()` 호출 및 이상 감지 시 알림 전송.
  - HTTP 스트리밍 `generate_frames(cap)`와 `/cctv` 엔드포인트 `stream_video()`로 실시간 영상 제공.
  - `upload_to_s3()`가 업로드 영상을 S3에 저장하고, `upload_video()`가 Colab 서버에 분석 요청 후 결과 JSON을 저장하며 이상 이벤트를 `Eventlog`로 기록.

- **기타**
  - 루트에는 `TTS_audio/`, `anomaly_detect_json/`, `video_data/` 등 생성 자원 디렉터리가 함께 관리된다.
  - `main.py` 외에 `anomalyDetect.py` 모듈은 독립 실행 진입점(`uvicorn.run`)을 갖추고 있으며 필요 시 단독 서비스로 구동 가능.
