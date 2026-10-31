택시팟 웹 클라이언트
대학생 택시 동승 매칭 서비스 택시팟의 웹 앱입니다.
API 명세서 3.10.0과 DB 테이블 설계서 14차를 기준으로 만들었습니다.
학생 앱(모바일 웹)과 운영자 콘솔이 하나의 프로젝트에 들어 있습니다.
서버 주소를 비워 두면 브라우저 안의 시연용 Mock 서버로 모든 흐름이 동작합니다.
서버 주소를 넣으면 팀의 Spring Boot 서버에 그대로 연결됩니다.
1. 빠르게 실행하기 (시연 모드)
Node.js 20 이상이 필요합니다.
npm install
npm run dev          # http://localhost:5173
화면 폭에 따라 보이는 모습이 달라집니다.
휴대폰: 앱이 전체 화면으로 열립니다.
데스크톱(폭 960px 이상): 서비스 소개와 iPhone 프레임 안의 앱이 함께 보입니다.
시연 계정
역할	이메일	비밀번호
학생 (인증 완료, 잔액 약 2만 원)	student@bu.ac.kr	taxipot123
운영자	admin@taxipot.kr	admin1234
로그인 화면의 시연 계정 버튼을 누르면 바로 로그인됩니다.
데이터를 처음 상태로 되돌리려면 마이 → 시연 데이터 초기화를 누르세요.
발표용 시나리오 (약 3분)
학생 계정으로 택시팟 만들기를 누릅니다.
출발지·도착지·시각·예상 요금을 입력하면 예치금이 묶입니다.
약 8초 뒤 시연 동승자가 자동으로 참여하고 채팅을 보냅니다.
20초쯤 뒤에는 이 동승자가 도착 확인을 합니다.
출발지 도착 확인을 누릅니다.
시연 모드에서는 출발지 근처 좌표로 GPS를 대신합니다.
탑승 시작을 누릅니다. 이때 카카오 T·Uber 딥링크가 나타납니다.
실요금 청구에서 결제내역 사진과 실제 요금을 올립니다.
운영자 계정으로 로그인해 실요금 청구 → 승인하고 정산을 누릅니다.
다시 학생 계정으로 돌아가 다음을 확인합니다.
정산 내역, 동승자 평가, 지갑 원장
원장에는 예치 해제 → 분담금 → 수수료 → 방장 정산금 순서로 기록됩니다.
이 밖에도 다음 흐름을 직접 확인할 수 있습니다.
회원가입: 시연 메일함에 인증코드가 표시됩니다.
학생 인증: 캡처를 제출하면 운영자가 승인합니다.
입장 요청: 참여 마감이 지난 방에서 보낼 수 있습니다.
출금 실패: 4,040원을 출금하면 실패 흐름이 나옵니다.
2. Spring Boot 서버에 연결하기
.env.example을 .env로 복사한 뒤 서버 주소를 넣습니다.
cp .env.example .env
VITE_API_BASE_URL=https://api.taxipot.kr
VITE_WS_URL=wss://api.taxipot.kr/ws       # 비우면 API 주소 + /ws
VITE_KAKAO_MAP_KEY=                        # 선택. 넣으면 실제 지도와 장소 검색을 사용
서버 쪽에서 확인할 것
화면은 명세 3.10.0을 그대로 따릅니다. 아래 항목이 서버와 맞아야 정상 동작합니다.
항목	필요한 설정
CORS	Authorization, Content-Type 헤더 허용. 아래 Date 노출도 필요합니다.
서버 시각	Access-Control-Expose-Headers: Date가 필요합니다. 참여 마감 판정을 서버 시각 기준으로 하기 때문입니다. 없으면 폰 시계를 사용합니다.
일시	Request·Response 모두 +09:00 오프셋을 사용합니다. 앱은 toKstIso()로만 보냅니다.
토큰	401 응답의 code로 구분합니다. UNAUTHORIZED이면 재발급 후 재시도하고, TOKEN_REVOKED이면 즉시 로그아웃합니다.
미수금	409 UNPAID_DEBT_EXISTS를 받으면 앱이 미수금 화면으로 이동합니다.
오류 형식	{ code, message, detail, timestamp } 형식이어야 합니다. 안내 문구에 detail 값을 사용합니다(부족액, 정지 해제 시각 등).
업로드	multipart 파트 이름을 맞춰야 합니다. 캡처는 captureImage + studentNo, 청구는 actualFare + receiptImage입니다.
이미지 조회	Bearer 토큰으로 fetch한 뒤 blob → createObjectURL로 표시합니다.
실시간	STOMP /ws를 사용합니다. CONNECT 헤더에 토큰을 담고, /topic/pots/{potId}를 구독해 CHAT_MESSAGE·POT_UPDATED를 받습니다.
엔드포인트별 대응 파일은 docs/API_MAPPING.md에 있습니다.
명세의 76개 API 중 72개를 화면과 연결했습니다.
나머지 4개는 다음과 같습니다.
자동 처리: /api/auth/refresh는 401 응답 때 자동으로 호출됩니다.
명세상 캡스톤 범위 외: PORTONE 충전 2개
선택 기능: 장애 복구 일괄 입력
3. 폴더 구조
src/
├── api/                 # 서버 계약과 통신
│   ├── types.ts         #   명세 스키마 전체 타입 (코드값·DTO)
│   ├── contract.ts      #   화면이 쓰는 API 계약 (REST·Mock 공통)
│   ├── http.ts          #   fetch 래퍼: 토큰 재발급, 오류 변환, 서버 시각 보정
│   ├── errors.ts        #   에러 코드 → 안내 문구 (detail 반영)
│   ├── endpoints/       #   도메인별 REST 구현
│   └── realtime/        #   STOMP 클라이언트
├── mock/                # 시연용 Mock 서버 (VITE_API_BASE_URL 이 비었을 때만 사용)
│   ├── db.ts            #   14개 테이블 구조 + 시드 데이터
│   ├── engine.ts        #   예치·환불·정산·미수금·정지·스케줄러 규칙
│   └── server.ts        #   계약 구현
├── config/              # 환경 변수, 정책 상수(수수료·마감 시간 등)
├── lib/                 # 시간(KST)·표시 형식·라벨·좌표·입력 검증
├── hooks/               # useAsync, useNow, useGeolocation, usePotRealtime 등
├── store/               # 세션·토스트 컨텍스트
├── routes/              # 접근 제어, 화면 틀
├── components/
│   ├── layout/          #   기기 프레임, 헤더, 탭바
│   ├── ui/              #   버튼, 입력, 시트, 배지, 별점, 사진 선택
│   ├── pot/             #   택시팟 카드, 경로, 좌석, 참여자 목록
│   └── map/             #   지도(카카오맵/약도), 장소 선택
├── screens/
│   ├── auth/            #   시작, 로그인, 가입 4단계, 학생 인증
│   ├── home/            #   지도 + 모집 목록 + 검색·정렬
│   ├── pot/             #   생성, 상세, 실요금 청구, 정산, 평가, 신고
│   ├── chat/            #   참여자 채팅 + 빠른 메시지
│   ├── mypots/          #   이용 기록
│   ├── wallet/          #   지갑, 원장, 미수금, 출금
│   ├── notifications/   #   알림
│   ├── profile/         #   마이, 정보 수정, 비밀번호, 계좌, 신고 내역, 탈퇴
│   └── admin/           #   운영자 콘솔: 인증 심사, 청구 승인, 회원·지갑, 신고, 통계
└── styles/              # tokens → base → layout → components → 화면별 CSS
database/
├── schema.sql           # MySQL 신규 설치용 DDL (설계서 14차, 14개 테이블)
└── migrations.sql       # 6차 → 14차 ALTER 이력
docs/API_MAPPING.md      # 엔드포인트 ↔ 코드 대응표
4. 명령어
명령	설명
npm run dev	개발 서버
npm run build	타입 검사 + 프로덕션 빌드 (dist/)
npm run preview	빌드 결과 미리보기
npm run typecheck	타입 검사만
npm run format	Prettier로 전체 정리 (.prettierrc.json, 한 줄 100자)
5. 배포
npm run build로 만든 dist/를 정적 호스팅에 올리면 됩니다.
주소 직접 접근(예: /pots/12)을 위한 설정은 이미 들어 있습니다.
Vercel: vercel.json
Netlify: public/_redirects
그 밖의 서버: 모든 경로를 index.html로 보내 주세요.
위치 확인(GPS)은 HTTPS에서만 동작합니다.
6. 정책 값 맞추기
수수료 1,000원, 예치 1.2배, 마감 5분·10분, 도착 반경 50m 같은 값은 한 곳에 모여 있습니다.
앱: src/config/policy.ts
서버: application.yml
서버 설정을 바꾸면 이 파일도 같이 바꿔 주세요.
앱의 값은 안내와 사전 검증에만 쓰이고, 최종 판정은 항상 서버가 합니다.
7. 알아 둘 점
Spring Boot 서버는 이 프로젝트에 없습니다.
이 저장소는 명세를 구현하는 클라이언트입니다.
Mock 서버는 시연과 화면 개발용입니다. 보안·동시성 보장은 실제 서버의 몫입니다.
충전(PORTONE): 명세 3.7.0에서 캡스톤 범위 외로 정해졌습니다. 시연 충전은 운영자 콘솔의 지갑 잔액 조정으로 합니다.
푸시 알림(FCM)
웹에서는 알림 목록을 30초마다 확인합니다.
앱 포장(Capacitor 등) 시에는 api.users.updateFcmToken()을 연결하면 됩니다.
지도: 카카오맵 키가 없으면 좌표를 투영한 약도를 보여 줍니다. 실제 도로와 다르다는 표시가 붙습니다.
카카오 T 딥링크: kakaot://taxi?... 스킴은 출시 전에 카카오 T 최신 앱에서 동작을 확인해 주세요. Uber는 공식 유니버설 링크를 사용합니다.
이전 v5 프로젝트: Supabase·토스페이먼츠 직접 결제 구조는 명세(지갑 예치·운영자 정산)와 맞지 않아 이번 버전에서 제외했습니다.
