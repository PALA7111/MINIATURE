# MINIATURE

나만의 작은 세계를 만들고, 그곳의 안내자와 잠시 머무는 감성형 AI 웹서비스입니다.

## 기술 스택

- Java 21 / Spring Boot 3.4.5 / Gradle 8.5
- Thymeleaf, HTML, CSS, JavaScript
- MySQL / Spring Data JPA
- Gemini 텍스트 생성 (`gemini-3.8-flash`) 및 Gemini TTS (`gemini-3.8-flash-tts`)

## 로컬 실행

1. MySQL(또는 호환 DB)을 실행하고 데이터베이스를 생성합니다.

```sql
CREATE DATABASE miniature CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

2. `.env.example`을 복사해 `.env` 파일을 만들고 DB 접속 정보, `GEMINI_API_KEY`, 사용할 포트(`SERVER_PORT`, 기본값 `8080`, 현재 로컬 `.env` 설정 `8081`)를 확인합니다.
3. Java 21 환경에서 프로젝트 폴더로 이동해 실행합니다.

```powershell
.\gradlew.bat bootRun
```

4. 브라우저에서 설정한 포트 주소(예: `http://localhost:8081` 또는 `http://localhost:8080`)를 엽니다.

Gemini API 키가 없거나 일시적 과부하가 발생하더라도 세계 텍스트는 선택한 설정에 맞춘 감성형 기본 응답으로 안전하게 동작합니다. TTS 음성 생성에는 유효한 `GEMINI_API_KEY`가 필요합니다.

## 미니어처 디오라마 영상 파일

`src/main/resources/static/videos/` 경로에 1920×1080 (16:9, 30fps, 8초 심리스 루프, H.264 `yuv420p`) 규격의 미니어처 영상 4종이 포함되어 있습니다.

- `winter_village.mp4` — 눈이 내리는 겨울 마을과 따뜻한 창가 불빛
- `fairy_forest.mp4` — 아침 햇살과 반딧불이가 머무는 요정의 숲
- `seaside_town.mp4` — 노을빛이 번지는 바닷가 작은 포구
- `rainy_city.mp4` — 잔잔한 빗방울과 가로등이 비치는 밤 골목

## API 테스트

서버 실행 및 MySQL 연결 확인, 세계 생성/저장/보관함/즐겨찾기/TTS/삭제 테스트는 [`CURL_TESTING.md`](CURL_TESTING.md)를 참고하세요. PowerShell에서는 별칭이 아닌 `curl.exe`를 사용합니다.

## 회원 인증 및 회원별 보관함

- MySQL `users` 테이블 기반 회원가입(`/signup`)·로그인(`/login`)·로그아웃(`/logout`)을 지원합니다(비밀번호 PBKDF2-HMAC-SHA256 솔트 해싱 적용).
- 모든 페이지 상단 헤더에서 로그인 상태에 따라 **로그인 / 회원가입** 또는 **{닉네임}님 / 로그아웃** 버튼이 표시됩니다.
- 보관함(`/archive`)과 즐겨찾기는 로그인한 회원 계정(`user_id`)별로 분리 저장되어, 다른 브라우저나 기기에서 로그인해도 내 보관함이 그대로 유지됩니다.
- 비로그인 상태에서 세계를 먼저 만든 뒤 `이 세계 저장하기`를 누르면 로그인/가입 후 해당 세계가 자동으로 회원 보관함에 귀속·저장됩니다.

## 보안

- 실제 `.env` 파일은 저장소에 커밋하지 않습니다.
- Gemini API 키는 백엔드 환경변수로만 관리하며 프론트엔드에 노출하지 않습니다.
- 익명 사용자 토큰은 브라우저 `localStorage`(`miniature_owner`)에 저장되며 생성/저장/보관함/TTS 요청에 동일하게 사용됩니다.
