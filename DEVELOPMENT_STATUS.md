# MINIATURE 개발 진행 현황

기준 문서: `PRD.md` (v1.0)

## 현재 구현 상태

| Phase | 내용 | 상태 | 비고 |
|---|---|---|---|
| 1 | Spring Boot / Gradle / Thymeleaf / MySQL 기본 구조 | 완료 | 로컬 MySQL 연결 및 `/api/health` 정상(`UP`) 확인 |
| 2 | 메인, 세계 설정, 로딩, 감상, 보관함, 오류 화면 | 완료 | 메인 쇼케이스·설정 미리보기·반응형 UI 적용 완료 |
| 3 | Gemini 텍스트 생성 (`gemini-3.8-flash`) | 완료 | 사고 토큰 분리·토큰 한도 상향·503 재시도 및 안전 폴백 검증 완료 |
| 4 | Gemini TTS (`gemini-3.8-flash-tts`) | 완료 | 공식 Interactions API `steps[].content[].data` 파싱 및 WAV 스트리밍 검증 완료 |
| 5 | 선택 조건별 미니어처 MP4 영상 4종 제작·삽입 및 매핑 | 완료 | 1920×1080 30fps H.264 심리스 루프 MP4 4종 삽입 및 재생 확인 |
| 6 | 회원 인증(가입/로그인/로그아웃) 및 회원별 보관함·즐겨찾기 | 완료 | `users` 테이블·세션 인증·회원별(`user_id`) 보관함 및 즐겨찾기 통합 검증 완료 |
| 7 | 배포 및 최종 검증 | 로컬 검증 완료 | 외부 운영 서버 배포만 남음 |

## 주요 오류 원인 분석 및 수정 내역

1. **Gemini TTS 음성 생성 실패 오류 수정 (`GeminiService`, `PcmWavUtil`)**
   - 공식 `/v1beta/interactions` 엔드포인트(`gemini-3.8-flash-tts`)는 오디오 데이터를 `steps[].content[].data` 구조로 반환하지만 기존 코드가 `output_audio.data`만 조회하여 항상 예외가 발생하던 문제를 수정했습니다.
   - 이미 RIFF/WAV 헤더가 포함된 오디오 응답의 전체 바이트 길이가 홀수일 때도 정상 처리되도록 `PcmWavUtil.toWav()`에서 `isWav()` 검사를 짝수 바이트 검사보다 먼저 수행하도록 수정했습니다.
2. **Gemini 텍스트 생성 JSON 절단 및 지연/503 오류 수정 (`GeminiService`)**
   - `gemini-3.8-flash` 호출 시 기존 `maxOutputTokens: 500` 설정이 내부 사고(thinking) 토큰에 소진되어 응답 JSON이 중간에 잘리던 문제를 `maxOutputTokens: 2048` 및 `thinkingConfig.thinkingBudget: 0` 적용으로 해결했습니다(응답 시간 약 7초로 단축).
   - `parts` 배열에서 `thought: true` 파트를 건너뛰고 실제 JSON 텍스트만 파싱하도록 보강했습니다.
   - 일시적 과부하(`HTTP 503`/`429`) 시 자동 재시도 후에도 외부 지연이 지속되면 장소·날씨·시간·분위기 조합에 맞춘 감성형 기본 세계관 데이터로 안전하게 폴백해 페이지 오류가 발생하지 않도록 개선했습니다.
3. **세계 상세 재방문 데이터 누락 및 저장 버튼 상태 오류 수정 (`World`, `WorldResponseDto`, `WorldService`)**
   - `worlds` 테이블에 `mood_text`, `voice_style` 컬럼을 추가해 보관함에서 다시 방문했을 때도 세계 분위기 문장과 캐릭터 말투가 그대로 표시되도록 수정했습니다.
   - `WorldResponseDto`에 `saved` 상태를 포함해 이미 저장된 세계를 열었을 때 버튼이 `저장됨`(비활성화)으로 정확히 표시되도록 수정했습니다.
4. **API 예외 처리 및 상태 코드 정합성 개선 (`GlobalExceptionHandler`, `WorldApiController`)**
   - 필수 쿼리 파라미터 누락(`MissingServletRequestParameterException`)이나 잘못된 토큰(`IllegalArgumentException`) 요청 시 `HTTP 500` 대신 `HTTP 400`을 일관되게 반환하도록 수정했습니다.
5. **프론트엔드 UX 및 디자인 보완 (`index.html`, `create.html`, `world.html`, `error.html`, `style.css`, JS)**
   - 메인 페이지(`index.html`): 4종의 미니어처 디오라마 영상을 미리 감상·전환할 수 있는 히어로 쇼케이스 프레임을 추가했습니다(PRD 8.1).
   - 세계 설정 페이지(`create.html`, `create.js`): 선택 항목에 따라 실시간으로 조합 문구와 매핑 배경 영상을 보여주는 **선택 결과 미리보기** 영역을 추가하고 `.option-card.selected` 스타일 누락을 수정했습니다(PRD 8.2).
   - 세계 감상 페이지(`world.html`, `world.js`): 세계 유형별 캐릭터 아이콘 적용, TTS 오류 발생 후 재시도 시 오류 메시지 초기화 및 `음성 다시 시도` 라벨 전환을 적용했습니다(PRD 5.4, 12).
   - 오류 페이지(`error.html`): `다시 세계 만들기` 재시도 이동 버튼을 명확히 배치했습니다(PRD 12).
6. **생성형 미니어처 디오라마 MP4 영상 4종 제작 및 삽입 (`src/main/resources/static/videos/`)**
   - PRD 4.1~4.3 규격(1920×1080, 16:9, 30fps, 8초 심리스 루프, H.264 High `yuv420p`, `+faststart`)을 충족하는 4개 영상을 제작해 삽입했습니다.
     - `winter_village.mp4`: 눈이 천천히 내리는 밤의 작은 마을과 따뜻한 창문 불빛
     - `fairy_forest.mp4`: 이끼 낀 숲속 오두막과 반딧불이·아침 햇살
     - `seaside_town.mp4`: 노을빛이 번지는 잔잔한 해안 포구와 작은 배
     - `rainy_city.mp4`: 빗방울이 내리는 밤 골목과 호박빛 가로등 반영

## 통합 검증 결과

- Gradle JUnit 단위 테스트(`PcmWavUtilTest`, `VideoMappingServiceTest`, `GeminiServiceTest`) 및 `bootJar` 빌드 통과
- `/api/health` DB 상태 `UP`, 모든 템플릿 페이지(`200 OK`), 4종 MP4 스트리밍(`200 OK` 및 `206 Partial Content`) 확인
- 세계 생성 → 감상 페이지 렌더링 → 보관함 저장 → 즐겨찾기 등록/해제 → Gemini TTS 음성 생성 및 WAV 파일 재생 헤더(`RIFF/WAVE`) → 캐시 재사용 → 삭제까지 전 과정 E2E 검증 완료
