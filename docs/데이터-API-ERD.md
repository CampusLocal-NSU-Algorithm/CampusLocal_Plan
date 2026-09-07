# 매물 데이터 조사 · 확정 API · 확정 ERD

> 결정 요약: **학교는 남서울대 고정, 집 검색 범위는 천안시 전체(동남구 44131 + 서북구 44133).**
> **"현재 매물"은 사용자 등록(직거래) 매물로 확보하고, 공공데이터 실거래가는 "시세가 적정한지" 검증·비교용으로 쓴다.**

---

## 1. "현재 매물" 데이터 조사 결과

### 결론: 무료·합법 공식 API로 "지금 나와 있는 일반 매물 목록"을 주는 곳은 없다.

| 소스 | 현재 매물 제공? | API | 사용 가능 여부 |
|---|---|---|---|
| 네이버 부동산 | O (플랫폼엔 있음) | **공식 API 없음** | 크롤링만 가능 → 약관·robots.txt 금지, 네이버가 크롤링 분쟁에서 승소한 이력(2026.2), 배포 서비스엔 부적합 |
| 직방 / 다방 / 피터팬 | O | **공식 오픈 API 없음** (다방은 신한은행 전세대출 제휴 API만) | 크롤링 금지 |
| 국토교통부 실거래가 (공공데이터포털) | X — **과거에 "계약된" 가격** | O (무료) | 전월세 신고제로 최근 1~2개월분은 거의 최신이지만, "지금 입주 가능한 매물"은 아님 |
| LH 공공임대 (행복주택·전세임대 등) | △ — 현재 "모집 공고" | O (무료, 공공데이터포털) | 물량 적고 상시 아님 → 보조 트랙으로만 |

### 그래서 매물 소스 모델 (확정)

1. **주력 = 사용자 등록 매물 (직거래 게시판형)**
   - 카카오 로그인한 사용자(재학생·졸업생·집주인·인근 공인중개사)가 "방 내놓기" 폼으로 등록
   - 주소는 카카오 주소검색으로 좌표 변환, 사진 최대 5장, 옵션·입주가능일·연락방법 입력
   - **매물은 등록 30일 후 자동 만료**(`expires_at`), 등록자가 "끌어올리기"로 연장, 거래되면 `closed`
     → 목록엔 항상 `status='open' AND expires_at > now()`만 노출 = **"없는 매물"이 안 뜬다**
   - 콜드스타트 대응: 발표 시연용으로 팀이 실제 천안 원룸 정보 15~20건 시드 등록 ("사용자 등록형 서비스"임을 명시)

2. **시세 검증 = 국토교통부 전월세 실거래가 (공공데이터 API)**
   - 매물 등록/조회 시 그 동네 실거래 시세(중앙값·사분위)를 함께 보여줌
     → "이 동네 원룸 월세 실거래 중앙값 35만(25~42만). 입력하신 40만은 평균보다 높음" 식 안내
   - 지도에 "동별 시세" 레이어로도 표시

3. **(선택) 공공임대 탭 = LH 공고 API** — 천안 행복주택·전세임대(대학생 유형) 모집 정보

이 구조면 **공공데이터 API 활용 요건 충족**(실거래가 + 상가정보 + 지오코딩 + 대중교통 + LH) + **현재 매물 존재** + **합법**.

---

## 2. 확정 외부 API 목록

### 공공데이터포털 (data.go.kr) — 무료, `serviceKey` 필요
| # | API | 오퍼레이션 | 파라미터 | 용도 |
|---|---|---|---|---|
| 1 | 국토교통부_연립다세대 전월세 실거래가 | `getRTMSDataSvcRHRent` | `serviceKey`, `LAWD_CD`, `DEAL_YMD`, `pageNo`, `numOfRows` | 시세(빌라) |
| 2 | 국토교통부_단독/다가구 전월세 실거래가 | `getRTMSDataSvcSHRent` | 〃 | 시세(원룸·단독) |
| 3 | 국토교통부_오피스텔 전월세 실거래가 | `getRTMSDataSvcOffiRent` | 〃 | 시세(오피스텔) |
| 4 | 소상공인시장진흥공단_상가(상권)정보 | `storeListInRadius` | `serviceKey`, `radius`, `cx`, `cy`, `type=json` | 매물 주변 인프라(반경 300m) |
| 5 | (선택) 한국토지주택공사_임대주택 공고/단지 조회 | 공고문·단지 조회 | 지역=천안 필터 | 공공임대 탭 |

- `LAWD_CD`(시군구 5자리): 천안시 **동남구 44131**, **서북구 44133** 둘 다 수집
- `DEAL_YMD`: 최근 **24개월** 각 월 반복 호출
- 응답 형식: 1~3번은 XML → `xml.etree.ElementTree`로 파싱

### 카카오 (developers.kakao.com) — 무료
| # | API | 엔드포인트 | 용도 |
|---|---|---|---|
| 6 | 카카오 로그인 (OAuth2) | `kauth.kakao.com/oauth/authorize`, `/oauth/token`, `kapi.kakao.com/v2/user/me` | 인증 |
| 7 | 카카오맵 JavaScript SDK | (JS 키, CDN) | 지도·마커·클러스터 |
| 8 | 카카오 로컬 - 주소→좌표 | `GET dapi.kakao.com/v2/local/search/address.json` | 매물 등록 시 지오코딩, 실거래가 지오코딩 |
| 9 | 카카오 로컬 - 좌표→행정구역 | `GET dapi.kakao.com/v2/local/geo/coord2regioncode.json` | 좌표 → 법정동코드(시세 매칭용) |
| 10 | (선택) 카카오 로컬 - 키워드 장소검색 | `/v2/local/search/keyword.json` | 매물 주변 편의시설 보조 검색 |

### 대중교통 / 도보
| # | API | 엔드포인트 | 용도 |
|---|---|---|---|
| 11 | ODsay 대중교통 길찾기 | `GET api.odsay.com/v1/api/searchPubTransPathT` (`SX,SY,EX,EY`) | 매물→남서울대 대중교통 소요시간·환승 (무료 1,000회/일) |
| 12 | (선택) 카카오모빌리티 보행자 길찾기 | 승인 필요 | 도보 시간. **미승인 시**: Haversine 직선거리 × 1.3 ÷ 4km/h 로 계산(외부 API 불필요) |

### 외부 API 아님
- **이미지 저장**: PythonAnywhere `media/uploads/` 폴더에 저장(매물당 5장, 등록 시 1200px 리사이즈). 초보 팀은 외부 스토리지 대신 로컬 폴더 권장.

### 키 관리 (`.env`, git 제외)
`DATA_GO_KR_SERVICE_KEY`, `KAKAO_REST_KEY`, `KAKAO_JS_KEY`, `KAKAO_CLIENT_SECRET`, `ODSAY_KEY`, `FLASK_SECRET_KEY`

---

## 3. 확정 ERD

### 엔터티

**users** — 카카오 로그인 사용자
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | PK | |
| kakao_id | BIGINT, UNIQUE, NOT NULL | 카카오 회원번호 |
| nickname | VARCHAR | |
| profile_image_url | VARCHAR | nullable |
| role | ENUM('student','landlord','agent','admin') | 기본 'student' |
| created_at | DATETIME | |

**search_profiles** — 저장된 추천 조건 (users 1:1)
| id PK / user_id FK UNIQUE / deposit_max INT(만원) / rent_max INT(만원) / commute_max_min INT / deal_type ENUM('wolse','jeonse','any') / room_types JSON / preferred_infra JSON / updated_at |

**buildings** — 건물(주소) 단위. 매물이 사라져도 후기·통학·인프라는 건물에 남는다
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | PK | |
| building_key | VARCHAR, UNIQUE | 도로명주소 정규화 or 좌표격자 해시 (동일 건물 묶기) |
| name | VARCHAR | nullable |
| address_road | VARCHAR | nullable |
| address_jibun | VARCHAR | |
| lat, lon | DOUBLE | |
| dong_code | CHAR(10) | 법정동코드 (시세 매칭) |
| created_at | DATETIME | |

**building_commute** — 남서울대까지 통학 (buildings 1:1, 최초 1회 계산·캐시)
| building_id FK PK / walk_min INT / walk_distance_m INT / transit_min INT / transit_summary VARCHAR / computed_at DATETIME |  (nullable 허용)

**building_nearby_places** — 건물 반경 300m 인프라 스냅샷 (buildings 1:N)
| id PK / building_id FK / category ENUM('cvs','cafe','food','pharmacy','hospital','gym','mart','laundry','bank') / name VARCHAR / distance_m INT / lat, lon DOUBLE |

**listings** — 사용자 등록 매물 (핵심 · users N:1, buildings N:1)
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | PK | |
| user_id | FK → users | 등록자 |
| building_id | FK → buildings | |
| title | VARCHAR | |
| room_type | ENUM('oneroom','villa','officetel','two_plus') | |
| deal_type | ENUM('wolse','jeonse') | |
| deposit | INT | 만원 |
| rent_monthly | INT | 만원 (전세면 0) |
| maintenance_fee | INT | 만원, nullable — 관리비 |
| area_m2 | FLOAT | nullable |
| floor | INT | nullable |
| build_year | INT | nullable |
| address_detail | VARCHAR | nullable, 동·호 등 (비공개 가능) |
| description | TEXT | |
| options | JSON | ['에어컨','세탁기','풀옵션','주차','반려동물',...] |
| available_from | DATE | nullable — 입주가능일 |
| contact_method | ENUM('in_app','phone','kakao_openchat') | |
| contact_value | VARCHAR | nullable |
| status | ENUM('open','reserved','closed') | 기본 'open' |
| view_count | INT | 기본 0 |
| created_at | DATETIME | |
| bumped_at | DATETIME | 끌어올리기 → 목록 정렬 기준 |
| expires_at | DATETIME | created_at + 30일. 지나면 목록 제외 |

**listing_images** — (listings 1:N)
| id PK / listing_id FK / image_url VARCHAR / sort_order INT |

**price_check_cache** — 동+유형별 실거래 시세 통계 (market_deals 집계, 배치 갱신). 매물 시세검증·지도 시세레이어
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | PK | |
| dong_code | CHAR(10) | |
| dong_name | VARCHAR | |
| room_type_group | ENUM('oneroom_villa','officetel') | |
| deal_type | ENUM('wolse','jeonse') | |
| deposit_median, rent_median | INT | |
| deposit_p25, deposit_p75, rent_p25, rent_p75 | INT | |
| sample_count | INT | |
| period_months | INT | 예: 6 |
| updated_at | DATETIME | |

**market_deals** — 국토교통부 실거래가 원본 (배치 수집)
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | PK | |
| deal_category | ENUM('yeonrip','danok','officetel') | 연립다세대/단독다가구/오피스텔 |
| deal_type | ENUM('wolse','jeonse') | |
| deposit | INT | 만원 |
| rent_monthly | INT | 만원 |
| area_m2 | FLOAT | |
| floor | INT | nullable |
| build_year | INT | nullable |
| sigungu_code | CHAR(5) | 44131 / 44133 |
| dong_name | VARCHAR | |
| dong_code | CHAR(10) | nullable |
| jibun | VARCHAR | nullable |
| lat, lon | DOUBLE | nullable (지오코딩 성공분만) |
| contract_ym | CHAR(6) | |
| contract_day | INT | nullable |
| source_hash | VARCHAR, UNIQUE | 중복수집 방지 |
| fetched_at | DATETIME | |

**favorites** — (users N:M listings)
| id PK / user_id FK / listing_id FK / memo VARCHAR nullable / created_at / **UNIQUE(user_id, listing_id)** |

**reviews** — 자취 후기 (users N:1, buildings N:1)
| 컬럼 | 타입 | 비고 |
|---|---|---|
| id | PK | |
| user_id | FK → users | |
| building_id | FK → buildings | |
| rating | INT (1..5) | |
| content | TEXT | |
| lived_year | INT | nullable — 거주 연도 |
| tags | JSON | nullable — ['채광좋음','방음약함','집주인친절',...] |
| created_at | DATETIME | |
| **UNIQUE(user_id, building_id)** | | 건물당 1인 1리뷰 |

**report_flags** — (선택) 허위매물 신고
| id PK / listing_id FK / user_id FK / reason VARCHAR / created_at |

**lh_notices** — (선택) LH 공공임대 천안 물량
| id PK / notice_type ENUM('happy_house','jeonse_lease','purchase_lease','national_rental') / title / region / apply_start DATE / apply_end DATE / detail_url / lat / lon / fetched_at |

### 관계 요약
```
users 1───N listings N───1 buildings 1───1 building_commute
users 1───1 search_profiles          buildings 1───N building_nearby_places
users 1───N favorites N───1 listings  buildings 1───N reviews
users 1───N reviews                   listings 1───N listing_images
market_deals ──(배치 집계)──▶ price_check_cache ──(dong_code)──▶ buildings
```

### "현재 매물" 보장 로직
- 목록/지도/추천 쿼리: `WHERE status='open' AND expires_at > NOW()` (정렬 `bumped_at DESC`)
- `expires_at` = `created_at` + 30일. 등록자가 하루 1회 "끌어올리기" → `bumped_at`, `expires_at` 갱신
- 거래 완료 → `status='closed'` (상세페이지엔 "마감" 표시, 목록엔 제외)
- 매일 만료 처리 스크립트 or 조회 시 필터로 처리

### 파생 데이터 계산 시점
- **매물 등록(POST /listings) 시**: 주소 지오코딩 → `buildings` upsert → 없으면 그 자리에서 ODsay 통학시간 + 상가정보 300m 조회해 `building_commute`·`building_nearby_places` 채움 (이미 있으면 재사용)
- **배치(주 1회)**: `market_deals` 수집 → `price_check_cache` 재집계
- 실시간 외부호출은 (a) 매물 등록 시 1건, (b) 카카오맵 JS, (c) 카카오 OAuth 뿐

---

## 4. 내부 라우트 (Flask)

| 메서드 · 경로 | 설명 | 로그인 |
|---|---|---|
| `GET /` | 랜딩 | |
| `GET /listings` | 목록+지도 (쿼리: `deal_type, deposit_max, rent_max, room_type, commute_max, q, sort, bbox`) | |
| `GET /listings/<id>` | 매물 상세 (+건물 통학·인프라·리뷰·시세비교) | |
| `GET /listings/new` · `POST /listings` | 방 내놓기 | O |
| `PATCH /listings/<id>` · `DELETE /listings/<id>` | 수정·상태변경·삭제 (본인) | O |
| `POST /listings/<id>/bump` | 끌어올리기 (하루 1회) | O |
| `GET /recommend` · `POST /recommend` | 조건 입력 → 점수 랭킹 | 저장은 O |
| `POST /favorites` · `DELETE /favorites/<listing_id>` | 즐겨찾기 토글 | O |
| `POST /buildings/<id>/reviews` · `PATCH /reviews/<id>` · `DELETE /reviews/<id>` | 리뷰 | O |
| `GET /api/price-check?lat=&lon=&room_type=&deal_type=` | 해당 동 실거래 시세 통계(JSON) | |
| `GET /mypage` | 내 조건·내 매물·즐겨찾기·내 리뷰 | O |
| `GET /auth/kakao` · `GET /auth/kakao/callback` · `POST /auth/logout` | 카카오 OAuth | |
| `GET /public-housing` | (선택) LH 공공임대 목록 | |

## 5. 추천 점수 (규칙 기반)

필터: `status='open'`, 예산·통학시간·유형 조건 통과분만.
점수(0~100 가중합):
- 가격 여유 35% — 예산 대비 매물가 여유 + 동네 실거래 중앙값 대비 저렴할수록 ↑
- 통학 30% — `min(walk_min, transit_min)` 짧을수록 ↑
- 인프라 20% — 건물 300m 내 선호 카테고리 시설 수
- 후기 15% — 건물 평균 별점 (없으면 중립 3.0)
