# Firebase 연동 수동 설정 가이드 (Phase 1-d-4-web)

`index.html` 의 POSE-CHECK 로그를 Firestore (`web_pose_logs`) 에 기록하기 위한 Firebase Web 연동 절차. Firebase Console 수동 작업이 필요하다.

## 1. Firebase Console 설정

### 1-1. 프로젝트 선택 / 생성

기존 `facial-sukje-app` 프로젝트를 공유해 쓰거나, 웹 전용 프로젝트를 신규로 생성한다.
- 기존 공유 방식은 모바일 앱 Firestore 와 동일 DB 인데 `web_pose_logs` 컬렉션만 별도 사용 → 간단.
- 신규 프로젝트는 완전 격리되지만 Console / Billing 이 2개로 늘어남.

### 1-2. 웹 앱 등록

Firebase Console → ⚙️ 프로젝트 설정 → 일반 → "내 앱" 섹션 → 웹 앱 추가 (`</>` 아이콘).
- 앱 닉네임: `facial-web` (임의).
- Firebase Hosting 설정은 **체크 해제** (Github Pages 사용 중).

등록 완료 후 표시되는 `firebaseConfig` 객체 6개 필드를 복사한다.

### 1-3. `index.html` 설정값 교체

`index.html` 하단 `<script type="module">` 블록의 다음 6개 값을 실제값으로 교체:

```js
const firebaseConfig = {
  apiKey: "TODO_API_KEY",              // ← 실제 값으로
  authDomain: "TODO_AUTH_DOMAIN",
  projectId: "TODO_PROJECT_ID",
  storageBucket: "TODO_STORAGE_BUCKET",
  messagingSenderId: "TODO_SENDER_ID",
  appId: "TODO_APP_ID"
};
```

> `apiKey` 는 Firebase Web SDK 상 **공개 식별자**이지만, Firestore 규칙으로 쓰기 범위를 통제한다 (아래 3장).

## 2. Authentication — 익명 로그인 활성화

Firebase Console → Authentication → Sign-in method 탭:
- "익명" Provider → **사용 설정**.

Settings → **승인된 도메인** 에 다음 도메인 추가:
- `jangkuk3.github.io` (Github Pages 프로덕션)
- `localhost` (로컬 테스트용, 기본 포함되어 있음)

## 3. Firestore 보안 규칙

Firestore Database → 규칙 탭. `web_pose_logs` 컬렉션에 **create 만** 허용하고 read/update/delete 는 전면 차단한다.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // (기존 users/... 규칙은 그대로 유지)

    // 웹 POSE-CHECK 로그: 익명 인증 필수, 쓰기 전용.
    match /web_pose_logs/{doc} {
      allow create: if request.auth != null
                    && request.auth.token.firebase.sign_in_provider == 'anonymous'
                    && request.resource.data.platform == 'web'
                    && request.resource.data.specVersion == 'v1.0';
      allow read, update, delete: if false;
    }
  }
}
```

게시 후 규칙이 활성화되기까지 최대 1분 소요.

## 4. 로컬 테스트

```bash
cd ~/StudioProjects/facial
python3 -m http.server 8888
```

브라우저로 `http://localhost:8888/` 접속 → 평가 1회 진행 → DevTools Console 에서 다음 로그 확인:
- `[POSE-CHECK] Firebase ready uid= ...` (익명 인증 성공)
- `[POSE-CHECK] movement=forehead attempt=1` ... (5동작 각 1회)

Firebase Console → Firestore → `web_pose_logs` 컬렉션에 5개 문서가 생성되면 성공.

## 5. 배포

로컬 확인 후 `feature/pose-check` → `main` 머지 + Github Pages 자동 배포.

## 6. 데이터 형태

각 `web_pose_logs/{doc}` 문서:

| 필드                    | 타입             | 설명                                                    |
| ----------------------- | ---------------- | ------------------------------------------------------- |
| `uid`                   | string           | 익명 uid (세션 식별, 영구 아님).                        |
| `sessionId`             | string           | `web_{timestamp}_{rand}`. 한 평가 세션 묶기용.          |
| `movement`              | string           | `forehead` / `eye` / `smile` / `snarl` / `pucker`.      |
| `timestamp`             | serverTimestamp  | Firestore 서버 시각.                                    |
| `pose_yaw`              | number? (deg)    | §3.2 근사.                                              |
| `pose_pitch`            | number? (deg)    | §3.3 근사.                                              |
| `pose_roll`             | number? (deg)    | §3.1.                                                   |
| `pose_ipd_px`           | number (px)      | §3.4.                                                   |
| `pose_bbox`             | map              | `{width:int, height:int}`.                              |
| `pose_center`           | map              | `{x:0~1, y:0~1}`.                                       |
| `rest_captured_at_ms`   | number? (ms)     | 측정 시작 (=`go(m)` 호출) 기준 baseline 캡처 시각.      |
| `peak_at_ms`            | number? (ms)     | 측정 시작 기준 동작 스냅샷 시각.                        |
| `score_total`           | int              | 해당 동작 1~5 등급.                                     |
| `score_detail`          | map              | `{ratio:double, grade:int}`.                            |
| `userAgent`             | string           | 브라우저 UA.                                            |
| `platform`              | string           | 항상 `"web"`.                                           |
| `specVersion`           | string           | 항상 `"v1.0"` (POSE_CHECK_SPEC).                        |

## 7. 장애 모드

- Firebase config 가 TODO 값인 채로 배포 → `initializeApp` 또는 `signInAnonymously` 가 실패하고 `[POSE-CHECK] auth failed` 경고만 남음. 평가/UI/CSV export 는 정상 동작.
- Firestore 규칙 거부 → `[POSE-CHECK] Firestore write failed` 경고. UI 영향 없음.
- 오프라인 → Firebase SDK 가 자동 재시도. 로컬 큐에 쌓임.

## 관련 문서

- `docs/POSE_CHECK_SPEC.md` — 계산 공식 공통 사양 (v1.0).
- 모바일 구현: `~/StudioProjects/facial_sukje/lib/services/pose_check.dart`.
