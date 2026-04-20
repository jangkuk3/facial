# POSE-CHECK 공통 사양 (v1.0)

- **버전:** v1.0
- **작성일:** 2026-04-20
- **상태:** 초안 (실기기 검증 반영 전)

## 1. 개요

### 1.1 목적

얼굴 측정 점수(FaceCoach Index, movement score 등)에 개입할 수 있는 **카메라 포즈 교란 변수**를 모든 플랫폼에서 **동일한 정의·동일한 단위**로 수집해, 점수 해석 시 포즈 영향을 분리해 분석하기 위함.

- 포즈 통제군 vs 변동군 간 점수 분산 차이를 정량화.
- 회차 간 점수 급변(폭발 회차)을 포즈 이상치로 설명 가능한지 검증.
- 향후 논문 Methods 섹션 초안으로도 활용.

### 1.2 적용 범위

| 구현 | 얼굴 엔진 | 비고 |
| --- | --- | --- |
| Flutter (Android) | Google ML Kit Face Mesh (468) | `Face.headEulerAngleX/Y/Z` SDK 값 사용 |
| Flutter (iOS) | Apple Vision (76 → ML Kit 468 매핑) | SDK yaw/pitch 제공 없음 → 랜드마크 기반 근사 |
| Web 앱 | MediaPipe Face Mesh (468) | SDK yaw/pitch 제공 없음 → 랜드마크 기반 근사 |

### 1.3 원칙

1. **비의료 지표**: POSE-CHECK 값은 점수에 영향을 주지 않으며, 해석용 메타데이터로만 사용.
2. **한 번 정의한 공식은 플랫폼 간 불변**: 플랫폼별 가용 데이터가 달라도 단위(degree/pixel/0~1)와 부호 규칙을 통일.
3. **V3 Movement 스코어링, Synkinesis 판정, `kUseV3NoSynkinesis` 로직은 건드리지 않는다.**

---

## 2. 측정 대상 변수 (8종)

### 2.1 머리 포즈 3축

| 변수 | 단위 | 부호 규칙 |
| --- | --- | --- |
| `yaw` | degree | 사용자 기준 우측이 양수 (오른쪽으로 돌림 > 0) |
| `pitch` | degree | 아래가 양수 (턱을 당김 > 0) |
| `roll` | degree | 시계방향(사용자 기준 오른쪽으로 기울임)이 양수 |

### 2.2 거리/크기 지표

| 변수 | 단위 | 설명 |
| --- | --- | --- |
| `ipd_px` | pixel | 좌/우 눈 중심 사이 유클리드 거리 |
| `bbox_width` | pixel | 얼굴 bbox 가로 |
| `bbox_height` | pixel | 얼굴 bbox 세로 |

### 2.3 위치 지표

| 변수 | 단위 | 설명 |
| --- | --- | --- |
| `center_x` | 0~1 | 얼굴 bbox 중심 x / 프레임 width |
| `center_y` | 0~1 | 얼굴 bbox 중심 y / 프레임 height |

좌표계: 프레임 (0,0) = **top-left**, x+는 오른쪽, y+는 아래. Apple Vision 에서 bottom-left 좌표로 내려올 경우 수집 직전 `y := 1 - y` 변환 후 기록 (기존 `resolveMlkitIndexFromParts` 규약과 동일).

### 2.4 타이밍 지표

| 변수 | 단위 | 기준점 |
| --- | --- | --- |
| `rest_captured_at_ms` | ms | 측정 시작(측정 버튼 tap) 기준, Rest baseline 확정(프레임 집계 완료) 시점 |
| `peak_at_ms` | ms | 측정 시작 기준, burst 대표 프레임(= 결과 score 산출 근거가 된 프레임) 시점 |

기준점은 **측정 시작 시각(t=0)** 으로 플랫폼·동작 공통.

---

## 3. 계산식 표준

### 3.1 Roll (플랫폼 공통, 가장 신뢰 가능)

좌/우 눈 중심 벡터의 각도.

```
left_eye_center  = (idx_33 + idx_133) / 2
right_eye_center = (idx_263 + idx_362) / 2
dx   = right_eye_center.x - left_eye_center.x
dy   = right_eye_center.y - left_eye_center.y
roll = atan2(dy, dx) * 180 / π
```

부호: 우측 눈이 좌측보다 아래로 내려가면(y 증가) `dy > 0` → roll > 0 → 사용자 기준 오른쪽 기울임. 위 표 부호 규칙과 일치.

### 3.2 Yaw (플랫폼별)

| 플랫폼 | 방법 |
| --- | --- |
| ML Kit (Android) | `Face.headEulerAngleY` 를 그대로 사용 (SDK 값; 좌측 돌림 음수 → 부호 규칙 맞춤 위해 값 그대로 사용해도 `yaw > 0` 이 우측 돌림으로 나옴을 실측 확인 필요) |
| MediaPipe (web) | 랜드마크 근사: `yaw ≈ atan2(nose_tip.x - face_center.x, ipd_px) * 180 / π`<br>`face_center.x = (idx_234.x + idx_454.x) / 2`, `nose_tip = idx_1` |
| Apple Vision (iOS) | MediaPipe와 동일한 근사식 사용 |

⚠️ ML Kit 값과 근사값 간 **±10% 이내 오차 허용**. 실기기 수집 데이터(양 플랫폼 동시 촬영)로 오프라인 회귀 → 근사값 → SDK 값 보정식 도출 후 v1.1 반영.

### 3.3 Pitch (플랫폼별)

| 플랫폼 | 방법 |
| --- | --- |
| ML Kit (Android) | `Face.headEulerAngleX` 사용 |
| MediaPipe (web) / Apple Vision (iOS) | 랜드마크 근사: `idx_10`(이마), `idx_152`(턱), `idx_1`(코끝) 삼각 관계로 얼굴 세로축 대비 코끝 상대 위치 계산. 잠정 공식:<br>`face_mid_y = (idx_10.y + idx_152.y) / 2`<br>`pitch ≈ atan2(idx_1.y - face_mid_y, ipd_px) * 180 / π` |

⚠️ Pitch 는 Yaw 보다 근사 오차가 크다. 실기기 데이터 회귀 필수. v1.1에서 보정식/허용 범위 확정.

### 3.4 IPD

```
left_eye_center  = (idx_33 + idx_133) / 2
right_eye_center = (idx_263 + idx_362) / 2
ipd_px = distance(left_eye_center, right_eye_center)
```

- ML Kit / MediaPipe: 픽셀 좌표 직접 사용.
- Apple Vision: `VNFaceLandmarkRegion2D.normalizedPoints` 는 bbox 기준 normalized. 프레임 픽셀 좌표로 역변환 후 거리 계산.
- 기존 `FaceAnalysisService.getIPD` 와 동일 계산식이므로 그 값을 POSE-CHECK 필드로도 그대로 재사용 가능.

### 3.5 BBox / Center

```
xs = [p.x for p in landmarks]
ys = [p.y for p in landmarks]
bbox_width  = max(xs) - min(xs)
bbox_height = max(ys) - min(ys)
center_x = ((min(xs) + max(xs)) / 2) / frame_width
center_y = ((min(ys) + max(ys)) / 2) / frame_height
```

- 랜드마크 집합: 해당 플랫폼의 **전체 랜드마크**(MediaPipe/ML Kit 468, Vision 76).
- `frame_width`, `frame_height` 는 현재 캡처된 이미지 크기(회전 보정 후 사용자가 보는 좌표계 기준).

---

## 4. 랜드마크 인덱스 매핑 (플랫폼 간 통일)

Flutter Android (ML Kit Face Mesh) 와 웹 MediaPipe 는 동일한 **468 MediaPipe 인덱스**를 사용. iOS Apple Vision 은 `lib/services/apple_vision_landmark_map.dart :: kVisionMap` 을 통해 파트+인덱스로 매핑.

| 의미 | MediaPipe / ML Kit (468) | Apple Vision (76) — part[index] |
| --- | --- | --- |
| 왼쪽 눈 바깥 | 33 | `leftEye[0]` |
| 왼쪽 눈 안쪽 | 133 | `leftEye[3]` |
| 오른쪽 눈 바깥 | 263 | `rightEye[0]` |
| 오른쪽 눈 안쪽 | 362 | `rightEye[3]` |
| 코끝 | 1 | `medianLine[6]` |
| 이마 (미드라인) | 10 | `medianLine[0]` |
| 턱 (미드라인) | 152 | `medianLine[8]` |
| 왼쪽 관자 | 234 | `faceContour[0]` |
| 오른쪽 관자 | 454 | `faceContour[16]` |

위 9개 인덱스는 모두 `kVisionMap` 에 이미 등록돼 있음 (2026-04-20 기준). iOS 전면카메라 좌우 반전 규약은 `apple_vision_service.dart` 의 기존 `isFrontCamera` 파라미터 흐름을 그대로 따른다.

⚠️ Apple Vision 의 내부 ordering (예: `leftEyebrow` 의 outer→inner 방향) 은 iOS SDK 버전에 따라 바뀐 전례가 있음. POSE-CHECK 는 가능하면 SDK 가 직접 제공하는 파트별 대표점만 사용하도록 위 9개로 한정하고, 다른 인덱스는 `v1.1` 에서 필요 시 추가.

---

## 5. Firestore 필드 스키마

- 경로: `users/{uid}/assessments/{assessmentId}` 문서 내 각 movement(`browRaise`, `eyeClosure`, `smile`, `snarl`, `lipPucker`) 별 **8개 추가 필드**.
- 모두 **optional (null 허용)**. 기존 문서는 그대로 읽히도록 `fromFirestore` 에서 null-safe 처리.
- 필드 명명: `{movementId}_pose_*` 접두사로 기존 필드(`{movementId}_effort`, `{movementId}_normMag`, …) 와 충돌 방지.

| 필드 | 타입 | 단위 | 비고 |
| --- | --- | --- | --- |
| `{movement}_pose_yaw` | number | degree | §3.2 |
| `{movement}_pose_pitch` | number | degree | §3.3 |
| `{movement}_pose_roll` | number | degree | §3.1 |
| `{movement}_pose_ipd_px` | number | pixel | §3.4 |
| `{movement}_pose_bbox` | map | `{width:int, height:int}` | §3.5 |
| `{movement}_pose_center` | map | `{x:float, y:float}` (0~1) | §3.5 |
| `{movement}_rest_captured_at_ms` | int | ms (측정 시작 기준) | §2.4 |
| `{movement}_peak_at_ms` | int | ms (측정 시작 기준) | §2.4 |

본 스키마는 `docs/ASSESSMENT_SCHEMA.md` 와 **동기 유지** — POSE-CHECK 구현 시 해당 문서에도 동일 필드 표를 반영한다.

---

## 6. 로그 포맷

실기기 디버깅용. 측정 완료(또는 burst 대표 프레임 확정) 시 **1 회 출력**. 값은 모두 소수점 1자리, 픽셀은 정수.

```
[POSE-CHECK] platform={ios|android|web} movement={name} attempt={n}
  yaw=X.X°  pitch=X.X°  roll=X.X°
  ipd_px=XXX  bbox=(WxH)  center=(X.XX, X.XX)
  rest_captured_at=+X.Xs  peak_at=+X.Xs
```

- `platform`: `ios` / `android` / `web` 고정.
- `attempt`: 같은 동작 내 burst 시도 번호 (현재 구현상 1 고정).
- `+X.Xs`: 측정 시작 기준 경과 시간. 내부 저장은 ms 정수(§5), 로그는 초 단위 소수 1자리.

---

## 7. 향후 분석 방법 (사용자 측정 데이터 수집 후)

1. **포즈 통제군 vs 변동군 점수 분산 비교**
   - 통제군: `|yaw| ≤ 5°`, `|pitch| ≤ 5°`, `|roll| ≤ 5°`, `0.4 ≤ center_x ≤ 0.6`.
   - 변동군: 위 조건 하나라도 위배.
   - 비교: 같은 동작·같은 사용자의 측정 회차 내 점수 표준편차.
2. **회귀 분석**
   - `score = β₀ + β_yaw·yaw + β_pitch·pitch + β_roll·roll + β_ipd·ipd_px + ε`
   - R² 값으로 "포즈로 설명되는 점수 변동 비율" 정량화.
   - 동작별 회귀계수 비교 → 어느 동작이 포즈에 가장 민감한지 파악.
3. **폭발 회차 포즈 이상치 검출**
   - 동일 사용자의 직전 5회 측정 대비 점수가 2σ 이상 편차인 회차.
   - 해당 회차의 POSE-CHECK 값이 개인 baseline 대비 outlier 인지 확인.
   - 둘 다 outlier → 포즈 원인 / 점수만 outlier → 동작 수행 원인.

---

## 8. 개정 이력

| 버전 | 날짜 | 변경 내용 |
| --- | --- | --- |
| v1.0 | 2026-04-20 | 최초 작성 (Flutter/웹앱 공유 기준, 실기기 검증 전) |
