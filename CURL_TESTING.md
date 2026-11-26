# MINIATURE API curl 점검

서버를 실행한 뒤 Windows PowerShell에서 `curl.exe`로 확인합니다.
PowerShell의 `curl` 별칭 대신 `curl.exe`를 사용합니다.

## 실행 전 준비

1. MySQL 서비스를 실행합니다.
2. MySQL에 `miniature` 데이터베이스를 생성합니다.

```sql
CREATE DATABASE miniature CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

3. 프로젝트 루트의 `.env.example`을 복사해 `.env`를 만들고 DB 접속 정보를 입력합니다.
4. Gemini 기능까지 테스트하려면 `.env`에 `GEMINI_API_KEY`를 입력합니다.
5. PowerShell에서 프로젝트 폴더로 이동해 서버를 실행합니다.

```powershell
.\gradlew.bat bootRun
```

서버가 `Started MiniatureApplication` 메시지를 출력한 뒤 아래 테스트를 진행합니다.
(`.env`에서 `SERVER_PORT=8081`과 같이 다른 포트를 지정한 경우 아래 명령의 `8080`을 해당 포트 번호로 바꿔 실행합니다.)

## 1. 서버 및 DB 상태

```powershell
curl.exe -i http://localhost:8080/api/health
```

기대 결과: HTTP 200, `status`와 `database`가 모두 `UP`.

## 2. 메인 페이지

```powershell
curl.exe -i http://localhost:8080/
```

기대 결과: HTTP 200, HTML 응답.

## 3. 세계 생성

아래 테스트 토큰은 실제 브라우저 토큰과 분리된 임시 값입니다.

```powershell
curl.exe -i -X POST "http://localhost:8080/api/worlds/generate" `
  -H "Content-Type: application/json" `
  --data-raw '{"locationType":"작은 마을","weatherType":"눈","timeType":"밤","moodType":"포근한","ownerToken":"curl-test-owner-0001"}'
```

기대 결과: HTTP 200 및 JSON 응답. 응답의 `worldId` 값을 기록합니다.
Gemini API 키가 없으면 서버의 개발용 기본 응답이 사용됩니다.

## 4. 저장 전 보관함 확인

```powershell
curl.exe -i "http://localhost:8080/api/worlds?ownerToken=curl-test-owner-0001"
```

기대 결과: HTTP 200 및 빈 배열 `[]`. 아직 저장 버튼을 누르지 않은 세계는 보관함에 표시되지 않습니다.

## 5. 세계 저장

아래 URL의 `{WORLD_ID}`를 3단계 응답의 `worldId` 숫자로 바꿉니다.

```powershell
curl.exe -i -X POST "http://localhost:8080/api/worlds/{WORLD_ID}/save" `
  -H "Content-Type: application/json" `
  --data-raw '{"ownerToken":"curl-test-owner-0001"}'
```

기대 결과: HTTP 200 및 `{"saved":true,...}`.

## 6. 저장 후 보관함 확인

```powershell
curl.exe -i "http://localhost:8080/api/worlds?ownerToken=curl-test-owner-0001"
```

기대 결과: HTTP 200 및 방금 저장한 세계가 포함된 JSON 배열.

## 7. 즐겨찾기 목록 조회

```powershell
curl.exe -i "http://localhost:8080/api/favorites?ownerToken=curl-test-owner-0001"
```

기대 결과: HTTP 200 및 즐겨찾기한 세계 ID 배열. 처음에는 `[]`가 반환됩니다.

## 8. 즐겨찾기 등록/해제

저장한 세계 ID를 사용합니다.

```powershell
curl.exe -i -X POST "http://localhost:8080/api/favorites/{WORLD_ID}?ownerToken=curl-test-owner-0001"
```

기대 결과: HTTP 200 및 `{"favorited":true,...}`. 해제는 같은 URL에 `-X DELETE`를 사용합니다.

## 9. 잘못된 선택값 검증

```powershell
curl.exe -i -X POST "http://localhost:8080/api/worlds/generate" `
  -H "Content-Type: application/json" `
  --data-raw '{"locationType":"없는 장소","weatherType":"눈","timeType":"밤","moodType":"포근한","ownerToken":"curl-test-owner-0002"}'
```

기대 결과: HTTP 400. 허용하지 않는 선택값은 Gemini API로 전달되지 않습니다.

## 10. TTS 생성

Gemini API 키를 `.env`에 설정하고, 생성에 성공한 세계 ID를 사용합니다.

```powershell
curl.exe -i -X POST "http://localhost:8080/api/worlds/{WORLD_ID}/tts?ownerToken=curl-test-owner-0001"
```

기대 결과: HTTP 200 및 `audioUrl` JSON. 반환된 URL을 확인합니다.

```powershell
curl.exe -sS -o tts-test.wav -w "HTTP %{http_code} | Content-Type: %{content_type}\n" "http://localhost:8080/audio/file/{AUDIO_FILENAME}"
```

기대 결과: HTTP 200 및 `Content-Type: audio/wav`. 프로젝트 폴더에 `tts-test.wav`가 저장되며, 파일을 재생해 음성을 확인합니다.

## 11. 삭제

```powershell
curl.exe -i -X DELETE "http://localhost:8080/api/worlds/{WORLD_ID}?ownerToken=curl-test-owner-0001"
```

기대 결과: HTTP 200 및 `{"deleted":true}`.

## 주의

- 1단계가 HTTP 503이면 먼저 MySQL 서비스, DB 이름, `.env`의 접속 정보를 확인합니다.
- 3단계가 HTTP 500이면 서버 콘솔 로그를 확인합니다. 응답에는 내부 예외 메시지가 노출되지 않습니다.
- 10단계는 유효한 Gemini API 키와 사용 가능한 TTS 모델이 필요합니다.
- TTS 응답은 Gemini Interactions API의 PCM 데이터를 WAV로 변환해 제공합니다.
- 테스트 후 11단계로 테스트 데이터를 삭제합니다.
