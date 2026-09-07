# 성능 기준선과 개선 실험

> 작성일: 2026-09-07 · 상태: accepted (코드 계측 결정, 운영 기준선 수집은 배포 후)

## 0. 범위

**결정 (2026-09-07)**: 최적화에 앞서 0번 계측을 추가한다. 기존 Amplitude와 서버 JSON 로그를 사용하며 새 분석 SDK·DB 테이블은 도입하지 않는다. 기본 OFF, QA 기준선은 샘플링 100%로 짧게 수집하고 실사용 확대 시 비용과 트래픽을 보고 비율을 낮춘다. 캐시·추천·연속일·인증·CORS preflight 캐시 정책은 변경하지 않는다.

최초 기준 코드는 API `067218d`, web `85f42b9`, mobile `abfa89b` 이후 계측 커밋이다. 이미 main에 반영된 홈 쿼리 왕복 개선, 폰트 경량화, 서버 `/me` 요청 단위 캐시는 되돌리지 않는다. 과거의 612KB 폰트/중복 호출 가정을 현재 기준선으로 사용하지 않는다.

| 저장소   | 변경                                                                                         |
| -------- | -------------------------------------------------------------------------------------------- |
| API      | Guard보다 앞선 HTTP middleware, 요청 ID/서버 처리 시간/DB driver I/O 로그                    |
| web      | API 응답 헤더 시간, 화면 데이터 반영, Web Vitals, 세션 동기화 단계, SSR `/me` 로그, 집계 CLI |
| mobile   | 셸 JS mount 시각/launch ID/앱 버전/OTA ID/네이티브 세션 조회 시간을 WebView에 주입           |
| umbrella | 본 측정·실험 가이드                                                                          |

## 1. 활성화와 롤백

API Render 환경변수:

```dotenv
PERFORMANCE_ENABLED=true
PERFORMANCE_SAMPLE_RATE=1
```

API 버전은 `RENDER_GIT_COMMIT` 자동 사용. 다른 환경에서는 `PERFORMANCE_RELEASE`에 SHA를 넣는다. `api_request` JSON이 서버 로그에 남는다. 샘플 요청만 `X-Request-ID`, `X-API-Release`, `Server-Timing: app;dur=...`를 반환한다. 브라우저에 이 헤더를 노출하되 `Access-Control-Max-Age`는 추가하지 않는다.

web Vercel 환경변수:

```dotenv
NEXT_PUBLIC_PERFORMANCE_ENABLED=true
NEXT_PUBLIC_PERFORMANCE_SAMPLE_RATE=1
PERFORMANCE_ENABLED=true
```

기존 `NEXT_PUBLIC_AMPLITUDE_API_KEY` 설정을 사용한다. 실제 키 이름은 web `.env.example`와 analytics 초기화 코드를 확인한다. 새 프로젝트/대시보드를 자동 생성하지 않는다. `NEXT_PUBLIC_*` 변경은 재빌드가 필요하다. web SHA·배포 환경은 Vercel 환경변수로 빌드에 포함된다. 로컬 프로덕션 빌드는 `NEXT_PUBLIC_PERFORMANCE_RELEASE`를 직접 지정한다. `PERFORMANCE_ENABLED`는 SSR 로그를 별도로 제어하며 브라우저 샘플링 비율을 따르지 않는다.

mobile은 JS 변경이며 앱 표시 버전은 올리지 않는다. OTA가 있으면 update ID, 내장 번들은 `EXPO_PUBLIC_PERFORMANCE_RELEASE`를 사용한다. 내장 번들 빌드 시 SHA를 지정하지 않으면 `embedded-unknown`으로 남으므로 그 표본은 버전 비교에서 제외한다. 구 mobile도 web/API 계측은 가능하지만 셸 시작 시간은 unknown이다. 새 API → web → mobile 순서로 검증하면 상관관계 확인이 쉽고, 구버전 조합도 동작한다. 머지/배포/OTA 발행은 별도 승인 후 한다.

계측 문제 시 환경변수를 false로 되돌리고 API 재시작/web 재빌드한다. 코드 롤백은 해당 PR revert로 하며 DB 마이그레이션은 없다. 개선 실험에서는 계측은 유지하고 해당 최적화 PR만 revert한다.

## 2. 지표의 정확한 의미

| 이벤트/필드                            | 의미와 주의사항                                                                                                                                                                                         |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Performance screen_start`             | 측정 시작. 하단 탭 클릭은 `trigger=tab`; 첫 문서는 navigation 시작; 다른 측정 화면 진입은 mount 기준                                                                                                    |
| `Performance screen_first_data`        | React 데이터 commit 후 두 번의 requestAnimationFrame. 최초 실제 데이터 표시의 근사치이며 실제 디스플레이 paint 보장은 아님                                                                              |
| `Performance screen_fresh_data`        | 성공적으로 API 확인한 데이터가 같은 방법으로 반영됨. 내용이 그대로여도 최신 확인 성공이면 집계                                                                                                          |
| `Performance screen_outcome`           | error/demo/abandoned. 정상 로딩 지연 분포에 섞지 않음                                                                                                                                                   |
| `Performance native_shell_home`        | JS 셸 mount → 첫 실제 홈 반영. OS 앱 프로세스 시작/네이티브 초기화는 포함하지 않음. native/web 연결은 wall clock이므로 기기 시간 변경에 영향받음                                                        |
| `Performance api_headers`              | fetch 시작 → 응답 헤더 수신. 전체 body 다운로드·JSON 파싱 시간, 앞선 Supabase 세션 조회는 포함하지 않음. SSE도 첫 응답 헤더까지                                                                         |
| `Performance web_vital`                | Next 내장 Web Vitals callback. LCP/FCP/INP/CLS/TTFB 등 브라우저가 지원·관측한 지표만 수집. CLS는 점수, 나머지는 ms                                                                                      |
| `Performance session_ready`            | 웹 쿠키를 서버에서 읽을 수 있는지 확인하는 전체 반복 시간·시도 수                                                                                                                                       |
| `Performance web_session_sync`         | mobile-entry에서 Supabase setSession 완료까지                                                                                                                                                           |
| `Performance mobile_entry_redirect`    | mobile-entry 문서 navigation 시작 → 홈 hard reload 직전                                                                                                                                                 |
| `web_server_performance` / `server_me` | SSR 실제 `/me` fetch 시작 → JSON 파싱 종료. 브라우저 이벤트가 아닌 Vercel 로그                                                                                                                          |
| `api_request.duration_ms`              | middleware 진입 → response finish/close. Guard·검증·처리·서버 응답 쓰기 포함. 사용자 기기 수신 완료는 아님                                                                                              |
| `api_request.headers_ms`               | 서버가 첫 헤더를 쓰기 전까지. 스트리밍 응답의 전체 처리 시간과 구분                                                                                                                                     |
| `db_count`, `db_ms`, `spans`           | Prisma pg driver의 queryRaw/executeRaw 횟수·시간. DB 네트워크/풀 대기 포함, PostgreSQL 순수 실행 시간은 아님. 트랜잭션 내 쿼리 포함, BEGIN/COMMIT 등 제어 명령·다른 pg 클라이언트·서버 외 요청은 미포함 |

DB 시간 합은 병렬 쿼리 때문에 요청 wall time보다 클 수 있다. `duration_ms - db_ms`를 앱 CPU 시간으로 해석하면 안 된다. SQL·파라미터를 기록하지 않아서 spans는 query/execute 구분과 소요 시간만 담으며 특정 SQL 식별은 별도 진단이 필요하다. 상세 spans는 요청당 64개까지, 총 count/ms는 계속 누적한다.

홈·놀이 소개·로드맵·리포트·상담 이력 화면을 계측한다. 놀이 진행 중 타이머로 리다이렉트되는 경우 소개 화면 성공으로 세지 않으며, 타이머 준비 시간은 이번 자동 계측 범위 밖이다. 홈 새로고침/자녀 변경/기타 링크의 클릭부터 시작하는 별도 journey는 아직 미포함이다. 하단 탭을 기준으로 비교한다.

TanStack Query는 이번에 도입하지 않는다. 현재 홈은 API 성공 시 first/fresh가 같은 시점이다. 이후 캐시 도입 때 `useScreenPerformance(route, source)`에 기존 캐시가 보이면 `cache`, 성공적인 재검증 후 `api`를 전달한다. 데이터 객체 참조 변경 여부가 아니라 query의 성공적인 fetch 상태로 구분해야 한다. 실패한 재검증은 `error`를 전달하고 캐시 화면은 유지할 수 있다.

Web Vitals는 SPA 탭마다 새 LCP가 아니라 **문서 navigation** 기준이다. INP/LCP 최종값은 상호작용/페이지 숨김 이후 보고될 수 있고 지원 안 되는 지표는 0이 아닌 미관측이다. Lighthouse 실험실 점수와 Amplitude 실사용 분포를 같은 값으로 비교하지 않는다.

## 3. 어디서 확인하나

1. Chrome DevTools Performance 녹화를 켜고 하단 탭을 클릭한다. Timings 트랙의 `yougabell:*` user timing에서 화면·API 구간을 본다. 메모리 누적을 막기 위해 기록 후 performance entry는 clear한다. 녹화된 trace를 저장한다.
2. Network에서는 개별 요청의 Waiting/Content Download/OPTIONS, 전송량, 응답의 request ID/Server-Timing을 확인한다. Network만으로 캐시 데이터가 실제 화면에 그려진 순간은 알 수 없다. 스크린샷 트랙과 화면 timing을 같이 본다.
3. Amplitude에서 `Performance screen_first_data`의 `duration_ms`를 route/trigger/버전/플랫폼/방문 유형별로 나누고 P50/P75/P95를 본다. `screen_fresh_data`는 별도로 본다. 플랜에서 백분위 집계를 제공하지 않으면 JSONL export 후 아래 CLI 사용.
4. `api_headers.request_id`로 API `api_request` 로그를 찾는다. SSR `/me`도 Vercel 로그의 동일 ID로 연결한다. 브라우저 `document_id`와 `journey_id`로 같은 화면 진입의 요청 수를 묶는다. API와 web 샘플링은 독립이므로 낮은 비율에서는 연결되지 않는 표본이 생긴다.
5. 챗봇 첫 토큰/전체 응답은 기존 Chat Response 이벤트로 계속 확인한다. 첫 토큰 수신과 타자기 애니메이션 완료는 다른 시점이다.

web 저장소에서 실행:

```bash
node scripts/summarize-performance.mjs /absolute/path/baseline.jsonl
node scripts/summarize-performance.mjs /absolute/path/after.jsonl
```

입력은 JSON 한 줄당 한 이벤트. Amplitude raw export (`event_type`, `event_properties`) 또는 API/Vercel의 JSON 본문을 지원한다. 로그 수집기가 붙인 타임스탬프 접두사/압축은 먼저 제거한다. CLI는 파싱 실패 줄 수를 출력하므로 누락 여부를 확인한다. 최근접 순위법 `ceil(p × N)`으로 P50/P75/P95를 계산하며 실패·중단을 지연 분포에서 제외하고 개수는 별도로 보여준다. 서로 다른 버전/플랫폼/방문 유형/측정 시작점을 합치지 않는다. 성공값 없는 그룹은 null, 0ms가 아니다.

## 4. 실험 절차와 완료조건

각 실험은 동일한 테스트 조건으로 **계측 ON 기준선 → 최적화 1개 → 계측 ON 재측정 → 유지 또는 revert → 필요 시 재측정** 순서로 한다. 계측 자체의 영향은 별도로 ON/OFF trace를 비교한다.

QA 실행 기록 템플릿:

| 항목         | 기록                                                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------------------------------------- |
| 실험 ID/시간 | 변경 PR, 측정 시각, 배포 완료 시각                                                                                        |
| 버전         | web/API SHA, appVersion, OTA update ID 또는 내장 SHA                                                                      |
| 기기         | OS/버전, 실제 기기 모델, WebView/Chrome 버전, 네트워크/스로틀링                                                           |
| 진입         | 첫 설치/첫 웹 저장소, 동일 세션 재방문, 탭 이동을 분리                                                                    |
| 데이터 상태  | QA 계정 별칭, 자녀 수/월령대, 완료 놀이 건수, 진행 중 놀이 유무, 리포트/알림/채팅 건수 등 해당 API가 처리할 데이터의 규모 |
| 캐시         | HTTP 캐시 ON/OFF, 앱 저장소 초기화 여부, 향후 Query 캐시 상태                                                             |
| 결과         | 표본 수, P50/P75/P95, 실패/중단 수, API 요청 수, DB count, 원본 trace/export 경로                                         |

데이터 상태 기록은 DB 덤프/개인정보 복사가 아니다. 데이터 양과 분기 조건이 다른 계정을 비교해 최적화 효과로 착각하지 않도록 QA 기록에만 남기는 요약이다. 실제 사용자/자녀 ID·이름·내용은 성능 이벤트에 넣지 않는다. 변경 전후 같은 스냅샷/계정을 쓰되 놀이 완료 등 테스트가 데이터를 바꾸면 복원 가능한 **테스트 환경에서만** 재설정한다.

`visit`은 계정 가입 이력이 아니라 브라우저 localStorage의 이전 관측 여부이며 sessionStorage에 고정해 hard reload 동안 유지한다. 저장소 차단 시 unknown, 초기화 시 다시 first가 될 수 있다. HTTP 캐시 hit/miss와는 별개다. 실제 첫 설치/복귀 세그먼트는 QA 실행 기록으로 확인한다. 첫 로그인/온보딩에 사용자가 머문 시간은 앱 초기 로딩으로 비교하지 않는다. native_shell_home은 `/mobile-entry`로 시작한 셸만 대상이다.

완료조건:

- 자동 테스트·타입 검사·프로덕션 빌드가 통과한다.
- 배포 후 실제 iOS/Android에서 탭 클릭→데이터 표시, 앱 시작→홈, API request ID 연결을 각각 확인한다.
- 샘플링 100% QA에서 정상/실패 요청을 발생시켜 이벤트 누락·중복, 데이터 노출이 없는지 확인한다. Guard 실패도 서버 로그에 남아야 한다.
- 실제 DB에 접속한 QA `/home`에서 driver count/spans가 0이 아닌지 확인한다. 단위 테스트는 DB 없이 동작하므로 실제 Prisma 엔진의 async context 연결은 배포 검증 대상이다.
- 조건별 30회 이상은 방향을 보는 탐색 표본으로만 사용한다. P95/리텐션 결론은 더 충분한 표본·기간을 확보하고 오류율/화면 정상성도 함께 본다. 리텐션 증가를 속도 개선만으로 인과 단정하지 않는다.
- 이번 코드 작업만으로 실사용 기준선을 측정했다고 보고하지 않는다. 계측 배포·실제 이벤트 확인·기준선 수집이 끝나야 0번 운영 완료다.

## 5. 개인정보와 비용

API 로그는 토큰/쿠키/SQL/파라미터/본문/사용자 ID를 기록하지 않는다. API 경로는 Express route template, 브라우저 경로는 정적 allowlist와 `:id`로 정규화하며 쿼리 문자열을 제거한다. 새로운 Amplitude properties에는 기술 메타데이터만 추가한다. 기존 Amplitude의 가명 사용자 ID·SDK 기본 메타데이터 정책은 그대로다.

로그 보존 기간/접근권한은 기존 Render/Vercel/Amplitude 운영 정책을 따르고 확장하지 않는다. 계측 100%는 무기한 기본 운영 권장이 아니다. 표본 수·이벤트 비용·계측 ON/OFF 부담을 확인하고 비율을 낮춘다. 화면 first/fresh와 API별 이벤트 때문에 기존보다 이벤트 수가 증가한다.
