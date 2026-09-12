# API 명세서 (API Spec)

> Base URL: `/api/v1`
> 인증 방식: 카카오 OAuth 로그인 후 발급되는 자체 JWT(Access Token)를 `Authorization: Bearer {token}` 헤더로 전달
> 모든 Response는 `application/json`. 아래 예시는 성공(200) 기준이며, 실패 시 공통 포맷은 문서 하단 참고.

---

## 1. 인증 (Auth)

### 1.1 카카오 로그인
`POST /api/v1/auth/kakao/login`

카카오 인가 코드로 로그인/최초 가입 처리 후 서비스 자체 토큰 발급.

**인증 필요**: 아니오

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| authorizationCode | string | Y | 카카오 OAuth 인가 코드 |
| redirectUri | string | Y | 프론트에서 사용한 redirect URI (카카오 토큰 교환용) |

**Response 예시**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": 12,
    "nickname": "안민영",
    "department": "컴퓨터소프트웨어학과",
    "profileImageUrl": "https://.../avatar.png",
    "isNewUser": false
  }
}
```

### 1.2 로그아웃
`POST /api/v1/auth/logout`

**인증 필요**: 예

**Request Body**: 없음 (Authorization 헤더의 토큰 기준으로 세션/리프레시 토큰 무효화)

**Response 예시**
```json
{ "message": "로그아웃되었습니다." }
```

### 1.3 토큰 재발급
`POST /api/v1/auth/token/refresh`

**인증 필요**: 아니오 (refreshToken 자체가 인증 수단)

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| refreshToken | string | Y | 발급받은 리프레시 토큰 |

**Response 예시**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

---

## 2. 지역 (Region)

### 2.1 지역 목록 조회
`GET /api/v1/regions`

02/02b 화면에서 사용. 학교로부터의 방향/거리와 매물 개수를 함께 반환.

**인증 필요**: 예

**Query Parameter**: 없음

**Response 예시**
```json
{
  "school": { "name": "남서울대학교", "latitude": 36.9430, "longitude": 127.1234 },
  "regions": [
    {
      "id": 1,
      "name": "성환",
      "description": "성환읍 · 남서울대학교 소재지",
      "propertyCount": 61,
      "distanceKm": 0,
      "direction": "here"
    },
    {
      "id": 2,
      "name": "평택",
      "description": "평택시",
      "propertyCount": 35,
      "distanceKm": 9.4,
      "direction": "north"
    },
    {
      "id": 3,
      "name": "직산",
      "description": "직산읍",
      "propertyCount": 27,
      "distanceKm": 5.5,
      "direction": "south"
    },
    {
      "id": 4,
      "name": "두정",
      "description": "두정동",
      "propertyCount": 58,
      "distanceKm": 10.5,
      "direction": "south"
    }
  ]
}
```

---

## 3. 매물 (Property)

### 3.1 지역별 매물 리스트 조회
`GET /api/v1/regions/{regionId}/properties`

03 화면(정렬/필터 포함)에서 사용.

**인증 필요**: 예

**Path Parameter**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| regionId | integer | Y | 지역 ID |

**Query Parameter**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| sort | string | N | `distance`(거리순, 기본값) \| `transit`(교통순) \| `price`(가격순) |
| amenities | boolean | N | true인 경우 주변 편의시설 요약 포함 |
| page | integer | N | 페이지 번호 (기본 1) |
| size | integer | N | 페이지당 개수 (기본 20) |

**Response 예시**
```json
{
  "region": { "id": 1, "name": "성환" },
  "totalCount": 61,
  "properties": [
    {
      "id": 101,
      "title": "성환동 행복빌라 3층",
      "thumbnailUrl": "https://.../thumb.jpg",
      "dealType": "MONTHLY",
      "deposit": 5000000,
      "monthlyRent": 350000,
      "roomType": "원룸",
      "walkMinutesToSchool": 8,
      "address": "충남 천안시 서북구 성환읍 성환리 123-4",
      "isFavorite": true,
      "rank": 1
    }
  ]
}
```

### 3.2 매물 상세 조회
`GET /api/v1/properties/{propertyId}`

04 화면에서 사용.

**인증 필요**: 예

**Response 예시**
```json
{
  "id": 101,
  "title": "성환동 행복빌라 3층",
  "photos": ["https://.../1.jpg", "https://.../2.jpg"],
  "dealType": "MONTHLY",
  "deposit": 5000000,
  "monthlyRent": 350000,
  "maintenanceFee": 50000,
  "roomType": "원룸",
  "areaM2": 26.4,
  "floorInfo": "3층/5층",
  "address": "충남 천안시 서북구 성환읍 성환리 123-4",
  "latitude": 36.9412,
  "longitude": 127.1245,
  "isFavorite": true,
  "region": { "id": 1, "name": "성환" },
  "complex": null
}
```
> `complex` 필드는 네이버 부동산 크롤링 시 단지 매물인 경우 단지정보 객체가 채워지며, 일반 원/투룸 매물은 `null`. 필드 상세는 `erd_spec.md`의 `property_complex` 테이블 참고. 현재 프론트 UI에는 미반영.

### 3.3 매물 주변 편의시설 조회
`GET /api/v1/properties/{propertyId}/amenities`

04 화면의 편의시설 미니맵에서 사용.

**인증 필요**: 예

**Response 예시**
```json
{
  "propertyId": 101,
  "amenities": [
    { "category": "CONVENIENCE_STORE", "name": "CU 성환점", "distanceMeters": 120 },
    { "category": "CAFE", "name": "메가커피 성환점", "distanceMeters": 210 },
    { "category": "GYM", "name": "성환 헬스클럽", "distanceMeters": 340 }
  ],
  "summary": { "convenienceStore": 2, "cafe": 2, "gym": 1 }
}
```

### 3.4 학교까지 경로/소요시간 조회
`GET /api/v1/properties/{propertyId}/route`

04(요약)·07(상세) 화면에서 사용.

**인증 필요**: 예

**Query Parameter**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| mode | string | N | `walk` \| `bus` \| `bike` \| `taxi`. 미지정 시 4개 이동수단 전체 요약 반환 |

**Response 예시 (mode=walk)**
```json
{
  "propertyId": 101,
  "mode": "walk",
  "durationMinutes": 8,
  "distanceMeters": 550,
  "steps": [
    { "description": "성환동 행복빌라 출발" },
    { "description": "성환로 따라 직진", "distanceMeters": 350 },
    { "description": "남서울대로에서 좌회전", "distanceMeters": 200 },
    { "description": "남서울대학교 정문 도착" }
  ]
}
```
> 경로 좌표(polyline)는 지도 API 연동 후 `route`에 포함 예정. MVP 초기에는 소요시간·거리·단계 텍스트만 제공 가능.

---

## 4. 즐겨찾기 (Favorite)

### 4.1 즐겨찾기 목록 조회
`GET /api/v1/favorites`

**인증 필요**: 예

**Query Parameter**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| regionId | integer | N | 특정 지역만 필터링 |
| sort | string | N | `distance` \| `transit` \| `price` |

**Response 예시**
```json
{
  "totalCount": 12,
  "properties": [
    {
      "id": 101,
      "title": "성환동 행복빌라 3층",
      "thumbnailUrl": "https://.../thumb.jpg",
      "monthlyRent": 350000,
      "walkMinutesToSchool": 8
    }
  ]
}
```

### 4.2 즐겨찾기 등록
`POST /api/v1/favorites`

**인증 필요**: 예

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| propertyId | integer | Y | 즐겨찾기할 매물 ID |

**Response 예시**
```json
{ "propertyId": 101, "isFavorite": true }
```

### 4.3 즐겨찾기 삭제
`DELETE /api/v1/favorites/{propertyId}`

**인증 필요**: 예

**Response 예시**
```json
{ "propertyId": 101, "isFavorite": false }
```

---

## 5. 매물 비교 (Compare)

### 5.1 매물 비교 데이터 조회
`POST /api/v1/properties/compare`

08에서 선택한 매물을 09 비교 테이블에서 보여주기 위해 사용. 조회와 동시에 비교 이력(`compare_history`)에 저장된다.

**인증 필요**: 예

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| propertyIds | integer[] | Y | 비교할 매물 ID 목록 (2~4개) |

**Response 예시**
```json
{
  "compareHistoryId": 55,
  "properties": [
    {
      "id": 101,
      "title": "성환동 행복빌라 3층",
      "thumbnailUrl": "https://.../thumb.jpg",
      "monthlyRent": 350000,
      "maintenanceFee": 50000,
      "roomType": "원룸 · 8평",
      "walkMinutesToSchool": 8,
      "busMinutesToSchool": 6,
      "nearbyAmenitySummary": "편의점 2 · 카페 2",
      "isFavorite": true,
      "bestFields": ["price"]
    }
  ]
}
```
> `bestFields`는 해당 매물이 비교군 내에서 최고값(★ 표시 대상)인 속성 목록.

---

## 6. 마이페이지 (My Page)

### 6.1 내 정보 조회
`GET /api/v1/users/me`

**인증 필요**: 예

**Response 예시**
```json
{
  "id": 12,
  "nickname": "안민영",
  "department": "컴퓨터소프트웨어학과",
  "profileImageUrl": "https://.../avatar.png",
  "kakaoLinked": true
}
```

### 6.2 최근 본 매물 조회
`GET /api/v1/users/me/recent-viewed`

**인증 필요**: 예

**Query Parameter**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| limit | integer | N | 최대 조회 개수 (기본 20) |

**Response 예시**
```json
{
  "properties": [
    { "id": 101, "title": "성환동 행복빌라 3층", "thumbnailUrl": "https://.../thumb.jpg", "viewedAt": "2026-09-10T09:12:00+09:00" }
  ]
}
```
> 매물 상세(3.2) 조회 시 서버에서 `recent_view`에 upsert하는 방식으로 적재.

### 6.3 비교했던 매물 이력 조회
`GET /api/v1/users/me/compare-history`

**인증 필요**: 예

**Response 예시**
```json
{
  "histories": [
    {
      "compareHistoryId": 55,
      "createdAt": "2026-09-10T10:00:00+09:00",
      "propertyTitles": ["성환동 행복빌라 3층", "성환역 앞 스위트홈", "성환 코너 자취방"]
    }
  ]
}
```

### 6.4 알림 설정 조회/변경
`GET /api/v1/users/me/notification-settings`
`PATCH /api/v1/users/me/notification-settings`

**인증 필요**: 예

**Request Body (PATCH)**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| newListingAlert | boolean | N | 관심 지역 신규 매물 알림 |
| priceDropAlert | boolean | N | 즐겨찾기 매물 가격 변동 알림 |

**Response 예시**
```json
{ "newListingAlert": true, "priceDropAlert": false }
```
> MVP 범위 외(알림 실제 발송은 미구현) — 설정값 저장까지만 지원.

---

## 9. 리뷰 (Review)

### 9.1 매물 리뷰 목록 조회
`GET /api/v1/properties/{propertyId}/reviews`

04 화면에서 사용. 평균 별점과 리뷰 개수를 함께 반환.

**인증 필요**: 예

**Query Parameter**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| page | integer | N | 페이지 번호 (기본 1) |
| size | integer | N | 페이지당 개수 (기본 20) |

**Response 예시**
```json
{
  "propertyId": 101,
  "averageRating": 4.5,
  "reviewCount": 12,
  "reviews": [
    {
      "id": 501,
      "userId": 12,
      "nickname": "김민수",
      "rating": 5,
      "content": "학교랑 가까워서 통학이 정말 편해요.",
      "createdAt": "2026-08-20T10:00:00+09:00"
    }
  ]
}
```

### 9.2 리뷰 작성
`POST /api/v1/properties/{propertyId}/reviews`

04 화면의 "리뷰 작성" 모달에서 사용. 매물당 사용자 1건만 작성 가능.

**인증 필요**: 예

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| rating | integer | Y | 별점 (1~5) |
| content | string | Y | 리뷰 내용 |

**Response 예시**
```json
{ "id": 501, "propertyId": 101, "rating": 5, "content": "학교랑 가까워서 통학이 정말 편해요.", "createdAt": "2026-08-20T10:00:00+09:00" }
```

### 9.3 리뷰 삭제
`DELETE /api/v1/reviews/{reviewId}`

본인이 작성한 리뷰만 삭제 가능 (본인 것이 아니면 403).

**인증 필요**: 예

**Response 예시**
```json
{ "message": "리뷰가 삭제되었습니다." }
```

---

## 10. 커뮤니티 (Community)

### 10.1 게시글 목록 조회
`GET /api/v1/community/posts`

11 화면에서 사용.

**인증 필요**: 예

**Query Parameter**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| category | string | N | `자유` \| `질문` \| `정보`. 미지정 시 전체 |
| page | integer | N | 페이지 번호 (기본 1) |
| size | integer | N | 페이지당 개수 (기본 20) |

**Response 예시**
```json
{
  "totalCount": 42,
  "posts": [
    {
      "id": 1001,
      "title": "성환 원룸 계약 전 체크리스트 공유해요",
      "authorNickname": "김민수",
      "category": "정보",
      "commentCount": 8,
      "createdAt": "2026-09-10T09:00:00+09:00"
    }
  ]
}
```

### 10.2 게시글 상세 조회
`GET /api/v1/community/posts/{postId}`

12 화면에서 사용. 댓글 목록을 함께 반환.

**인증 필요**: 예

**Response 예시**
```json
{
  "id": 1001,
  "title": "성환 원룸 계약 전 체크리스트 공유해요",
  "content": "성환 원룸 처음 구할 때 등기부등본이랑...",
  "authorNickname": "김민수",
  "category": "정보",
  "createdAt": "2026-09-10T09:00:00+09:00",
  "comments": [
    { "id": 2001, "authorNickname": "정하늘", "content": "감사해요! 저도 참고할게요.", "createdAt": "2026-09-10T10:00:00+09:00" }
  ]
}
```

### 10.3 게시글 작성
`POST /api/v1/community/posts`

13 화면에서 사용.

**인증 필요**: 예

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| title | string | Y | 제목 |
| content | string | Y | 본문 |
| category | string | N | `자유` \| `질문` \| `정보` |

**Response 예시**
```json
{ "id": 1001, "title": "성환 원룸 계약 전 체크리스트 공유해요", "category": "정보", "createdAt": "2026-09-10T09:00:00+09:00" }
```

### 10.4 게시글 삭제
`DELETE /api/v1/community/posts/{postId}`

본인이 작성한 게시글만 삭제 가능 (본인 것이 아니면 403).

**인증 필요**: 예

**Response 예시**
```json
{ "message": "게시글이 삭제되었습니다." }
```

### 10.5 댓글 작성
`POST /api/v1/community/posts/{postId}/comments`

12 화면의 댓글 입력창에서 사용.

**인증 필요**: 예

**Request Body**
| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| content | string | Y | 댓글 내용 |

**Response 예시**
```json
{ "id": 2001, "postId": 1001, "content": "감사해요! 저도 참고할게요.", "createdAt": "2026-09-10T10:00:00+09:00" }
```

### 10.6 댓글 삭제
`DELETE /api/v1/community/comments/{commentId}`

본인이 작성한 댓글만 삭제 가능 (본인 것이 아니면 403).

**인증 필요**: 예

**Response 예시**
```json
{ "message": "댓글이 삭제되었습니다." }
```

---

## 11. 공통 에러 응답 포맷

```json
{
  "error": {
    "code": "PROPERTY_NOT_FOUND",
    "message": "존재하지 않는 매물입니다."
  }
}
```

| HTTP Status | 사용 예 |
|---|---|
| 400 | 잘못된 요청 파라미터 (예: propertyIds 0개 또는 5개 이상) |
| 401 | 토큰 없음/만료 |
| 403 | 권한 없음 |
| 404 | 리소스 없음 (매물/지역 등) |
| 500 | 서버 내부 오류 |

---

## 12. 추후 고도화 시 고려
- 네이버 부동산 실시간/증분 크롤링 스케줄러 및 매물 변경 감지(가격 변동, 매물 만료 처리)
- 지도 API(카카오맵/네이버지도) 연동 후 실제 경로 polyline·정확 좌표 기반 거리 계산
- 매물 리스트/상세 응답 캐싱 전략
- 커뮤니티 게시글/댓글 신고·모더레이션
- 리뷰 사진 첨부
