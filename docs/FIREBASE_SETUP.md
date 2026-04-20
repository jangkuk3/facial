# Firebase 연동 설정 (Phase 1-d-4-web) — 완료 상태 문서

`index.html` 의 POSE-CHECK 로그를 Firestore (`web_pose_logs`) 에 기록하기 위한 Firebase Web 연동. **Firebase Console 사전 준비 완료** — 본 문서는 설정 요약 / 향후 변경 시 참조용.

## 1. 프로젝트 · 웹 앱 등록 (완료)

- **Project ID:** `facial-sukje-app` (모바일 앱과 동일 프로젝트 공유)
- **웹 앱 닉네임:** `FaceCoach Web (POSE-CHECK)`
- **`firebaseConfig`** 은 `index.html` 하단 `<script type="module">` 블록에 하드코딩되어 있다.
  - `apiKey` 는 Firebase Web SDK 상 **공개 식별자** — Firestore 규칙으로 쓰기 범위를 통제 (아래 3장).
  - 향후 key rotation 이 필요하면 Firebase Console → 프로젝트 설정 → 일반 → 내 앱 > SDK 구성 에서 새 값 복사 후 `index.html` 교체.

## 2. Authentication — 익명 로그인 (활성화 완료)

- Authentication → Sign-in method 탭 → "익명" Provider **사용 설정됨**.
- Settings → 승인된 도메인: `jangkuk3.github.io`, `localhost`.

## 3. Firestore 보안 규칙 (게시 완료)

`web_pose_logs` 컬렉션에 **create 만** 허용하고 read/update/delete 는 전면 차단한다.

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
