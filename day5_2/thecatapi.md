# The Cat API 사용법

> 출처: https://docs.thecatapi.com/ (요청하신 `https://thecatapi.com/API`는 404였고, `developers.thecatapi.com`이 위 주소로 301 리다이렉트됨)
> 조사일: 2026-11-09
> 아래 "확인 필요" 표시는 공식 문서 원문이 아니라 요약본에서 얻은 내용이거나, 문서끼리 서로 다른 내용이니 실제 호출로 검증할 것.

## 1. 기본 정보

| 항목 | 내용 |
|---|---|
| Base URL | `https://api.thecatapi.com/v1` |
| 인증 | 모든 요청에 `x-api-key` 헤더 |
| API 키 발급 | https://thecatapi.com/signup (이메일로 발송) |
| 응답 형식 | JSON |

```bash
curl -H "x-api-key: YOUR-API-KEY" https://api.thecatapi.com/v1/breeds
```

- 홈페이지 샘플 코드도 같은 Base URL과 헤더(`"x-api-key": "DEMO-API-KEY"`)를 쓴다. 홈페이지의 샘플 요청:
  `https://api.thecatapi.com/v1/images/search?size=med&mime_types=jpg&format=json&has_breeds=true&order=RANDOM&page=0&limit=1`
  (여기서 `order=RANDOM`, `size`, `mime_types`가 쓰인다. 위 `RAND`/`RANDOM` 불일치는 `RANDOM`이 맞을 가능성이 높다.)
- 문서에는 `api_key` 쿼리 파라미터가 나오지 않는다. 헤더 방식만 사용한다.
- API 키는 브라우저 코드에 그대로 노출된다. 공개 배포 시 키 노출에 주의한다.

## 2. 요금제와 제한

- 홈페이지(https://thecatapi.com) 요금표: Free 월 10,000회 / Startup $99 월 100,000회 / Enterprise 무제한(문의). 홈페이지에는 분당 제한이 적혀 있지 않다.
- 아래 "분당 10회"는 `docs.thecatapi.com` 안내 문구에서 온 값이다. 월 한도와 분당 한도가 둘 다 적용되는 것으로 보이지만 두 출처가 같은 내용을 말하는지는 확인되지 않았다 ← 확인 필요
- **Free**: 월 10,000회, 분당 10회 요청 제한(문서 기준). 품종 응답은 핵심 필드(name, temperament, origin, life_span, description, image)만 제공.
- **Startup**: 더 높은 요청 한도, 상업 라이선스, 전체 품종 데이터(1~5점 특성 점수, 불리언 속성), 번역(`lang`), 품종별 facts.
- **Enterprise**: 사진 분석, 건강 기록, 계산기, 건강 팁 등.
- `limit`은 API 키가 있을 때 1~100. 키가 없을 때의 한도는 문서에 숫자가 없다.
- 업로드: 월간 업로드 쿼터가 있고, 초과하면 400. 사진 50MB, 동영상 200MB 제한.
- 남은 쿼터 조회: `GET /accounts/quota` → `quota`, `usage`, `remaining`
- 사용 가능한 기능 조회: `GET /accounts/features`
- 속도 제한 관련 응답 헤더는 문서화되어 있지 않다.

## 3. 주요 엔드포인트

### 3.1 이미지 (Images)

#### `GET /images/search` — 고양이 사진 검색 (가장 많이 쓰임)

```bash
curl -H "x-api-key: YOUR-API-KEY" \
  "https://api.thecatapi.com/v1/images/search?limit=2&breed_ids=abys&include_breeds=1"
```

| 파라미터 | 설명 |
|---|---|
| `limit` | 개수. 1~100 (키 있을 때), 기본 1 |
| `page` | 페이지 번호 (0부터) |
| `order` | `ASC`, `DESC`, `RAND` (문서) / `RANDOM` (OpenAPI 요약) ← 확인 필요 |
| `has_breeds` | `1`이면 품종 정보가 있는 사진만 |
| `breed_ids` | 품종 ID, 쉼표로 여러 개 (예: `abys,beng`) |
| `include_breeds` | `1`이면 응답에 품종 정보 포함 |
| `include_categories` | `1`이면 응답에 카테고리 포함 |
| `sub_id` | 사용자 구분용 ID |
| `mime_types`, `size`, `category_ids` 등 | 확인 필요 (요약본에만 있고 원문 미확인) |

응답 예시:

```json
[
  {
    "id": "KWdLHmOqc",
    "url": "https://cdn2.thecatapi.com/images/KWdLHmOqc.jpg",
    "width": 3114,
    "height": 2609,
    "breeds": []
  }
]
```

#### 그 밖의 이미지 엔드포인트

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/images/{id}` | 이미지 1개 조회 (예: `/images/KWdLHmOqc`) |
| GET | `/images` | 내가 업로드한 이미지 목록 (`sub_id` 선택) |
| DELETE | `/images/{id}` | 업로드한 이미지 삭제 |
| GET | `/images/{id}/labels` | 업로드 이미지의 AI 품종/카테고리 라벨 |
| POST | `/images/upload` | 이미지 업로드 (multipart, `file=@/path/cat.jpg`) |
| POST | `/images/upload-sync` | 분석 완료까지 기다리는 업로드 (`auto_crop` 기본 true) |
| GET | `/images/{id}/status` | 업로드 심사 상태 |

- 업로드 상태값: `to_be_reviewed`, `clean`, `unsafe`, `flagged`, `does_not_contain_species`, `labelled`
- `upload-sync`의 호스트가 문서마다 다르다. `upload.thecatapi.com`(이미지 문서) / `api.thecatapi.com`(레퍼런스) ← 확인 필요
- 공개 요청: `POST/DELETE /images/{imageId}/public-requests`, `GET /images/{imageId}/public-requests/status`

### 3.2 품종 (Breeds)

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/breeds` | 품종 목록 |
| GET | `/breeds/search?q=siamese` | 품종 이름 검색 (`q` 1~30자, 별칭도 매칭) |
| GET | `/breeds/{id}` | 품종 1개 (예: `abys`) |
| GET | `/breeds/{id}/colours` | 품종의 색/무늬 변형 |
| GET | `/breeds/breed-groups` | 품종 그룹 목록 (문자열 배열) |

`/breeds` 쿼리 파라미터:

| 파라미터 | 설명 |
|---|---|
| `limit` | 1~1000, 기본 1000 |
| `page` | 0부터, 기본 0 |
| `order` | `ASC`, `DESC`, `RANDOM` |
| `breed_groups` | 쉼표 구분, 대소문자 무시 (예: `Short-haired,Rex`) |
| `exclude_breed_groups` | 제외할 그룹 |
| `extinct` | 멸종 품종 포함 여부 (기본 false) |
| `lang` | 번역 언어 (예: `it`, Startup 이상) |
| `species_id`, `venom_code` | 기타 필터 |

```bash
curl -H "x-api-key: YOUR-API-KEY" "https://api.thecatapi.com/v1/breeds?breed_groups=Short-haired,Rex"
curl -H "x-api-key: YOUR-API-KEY" "https://api.thecatapi.com/v1/breeds/search?q=siamese"
```

품종 응답 예시 (핵심 필드만):

```json
{
  "id": "abys",
  "name": "Abyssinian",
  "species_id": "1",
  "life_span": "14-17",
  "temperament": "Active, Energetic, Independent, Intelligent, Gentle, Curious, Playful",
  "origin": "Egypt",
  "country_code": "EG",
  "breed_group": "Short-haired",
  "reference_image_id": "KWdLHmOqc",
  "image": {
    "id": "KWdLHmOqc",
    "url": "https://cdn2.thecatapi.com/images/KWdLHmOqc.jpg",
    "width": 3114,
    "height": 2609
  },
  "adaptability": 3,
  "affection_level": 3,
  "rex": false,
  "hypoallergenic": false
}
```

- 전체 응답에는 `description`, `history`, `alt_names`, `wikipedia_url`, `cfa_url` 등과 `indoor`, `lap`, `hairless`, `rare` 같은 불리언이 추가로 포함된다.
- 고양이 품종 그룹: Short-haired, Semi-longhair, Long-haired, Medium-hair, Rex, Hairless, Hybrid 등.
- `adaptability`, `affection_level` 같은 점수/불리언은 Free 플랜에서는 오지 않을 수 있다.

### 3.3 투표 (Votes)

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/votes` | 투표 목록 (`limit` 1~100 기본 10, `page`, `order`, `sub_id`) |
| POST | `/votes` | 투표 생성 |
| GET | `/votes/{vote_id}` | 투표 1개 조회 |

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR-API-KEY" \
  -d '{"image_id": "KWdLHmOqc", "value": 1}' \
  https://api.thecatapi.com/v1/votes
```

- 투표는 삭제할 수 없다. 취소하려면 반대 `value`로 다시 투표한다.
- `value`의 허용 값(보통 1 = 좋아요, 0 = 싫어요)은 조사한 범위에서 명시되지 않았다. ← 확인 필요

### 3.4 즐겨찾기 (Favourites)

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/favourites` | 목록 (`limit`, `page`, `order`, `image_id`, `sub_id`, `attach_image` 등) |
| POST | `/favourites` | 추가. 본문 `{"image_id": "KWdLHmOqc"}` (`sub_id` 선택) |
| GET | `/favourites/{id}` | 1개 조회 |
| DELETE | `/favourites/{id}` | 삭제 |

### 3.5 카테고리 (Categories)

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/categories` | 카테고리 목록 |
| GET | `/categories/{id}` | 카테고리 1개 |

### 3.6 고양이 상식 (Facts)

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/facts` | 목록 (`limit` 1~100 기본 10, `offset`, `breed_id`, `species_id`, `category`, `include_sources`, `lang`) |
| GET | `/facts/random` | 랜덤 1개 |
| GET | `/facts/breed/{breedId}` | 품종별 목록 (예: `/facts/breed/abys?limit=5`) |
| GET | `/facts/breed/{breedId}/random` | 품종별 랜덤 |
| GET | `/facts/species/{speciesId}/random` | 종별 랜덤 |
| GET | `/facts/{id}` | 1개 조회 |

- facts는 `page` 대신 `offset`을 쓴다. 품종별 facts는 Startup 이상이다.

### 3.7 기타

- 건강 팁: `GET /health-tips`, `/health-tips/breed/{breedId}`, `/health-tips/{id}` (Enterprise)
- 품종 금지 국가: `GET /breeds/bans/countries`, `/breeds/bans/country`, `/breeds/bans/breed/{breedId}`
- 계정: `GET /accounts/features`, `/accounts/quota`, `/accounts/capabilities`
- Sub ID: `GET /sub-ids`
- 반려동물/사진 분석/계산기/웹훅 (`/pets`, `/calculators`, `/webhooks` 등)은 Enterprise 유료 기능이라 웹페이지 기본 용도와 거리가 멀어 생략한다.

## 4. 페이지네이션 / 정렬 규칙

- `limit`: 개수 (보통 1~100, 기본 10)
- `page`: 0부터 시작하는 페이지 번호
- `order`: `ASC` / `DESC` / `RANDOM`
- facts, health-tips는 `page` 대신 `offset`

## 5. 오류 처리

오류 응답 형식:

```json
{
  "statusCode": 404,
  "timestamp": "2026-10-06T03:03:08.534Z",
  "path": "/v1/pets/doesnotexist",
  "message": "Pet with id \"doesnotexist\" not found",
  "error": "Not Found"
}
```

`message`가 아니라 `statusCode`와 (있다면) `code` 필드로 분기한다.

| 상태 | 의미 | 재시도 |
|---|---|---|
| 400 | 잘못된 요청, 월 업로드 쿼터 초과 | X |
| 401 | 비활성 계정의 키 | X |
| 403 | 키 없음/잘못된 키, 해당 플랜에 없는 기능 | X |
| 404 | ID가 없거나 삭제됨 | X |
| 409 | 이미 존재 | X |
| 415 | 지원하지 않는 업로드 형식 | X |
| 422 | 요청은 유효하나 처리 불가 | X |
| 429 | 요청 과다 | O (백오프) |
| 500 | 서버 오류 | O (백오프) |
| 503 | 일시적 사용 불가 | O (백오프) |

재시도는 429/500/503만 한다. 1초 → 2초 → 4초 → 8초 + 무작위 지연으로 최대 4~5회 시도한다.

## 6. 이 프로젝트(HTML + JS)에서의 사용 예

```html
<img id="cat" alt="고양이" class="rounded-lg w-full" />
<script>
  const API_KEY = 'YOUR-API-KEY';

  async function loadCat() {
    try {
      const res = await fetch(
        'https://api.thecatapi.com/v1/images/search?limit=1&include_breeds=1',
        { headers: { 'x-api-key': API_KEY } }
      );
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const [cat] = await res.json();
      document.getElementById('cat').src = cat.url;
    } catch (err) {
      console.error('고양이 불러오기 실패:', err);
    }
  }

  loadCat();
</script>
```

주의:
- Free 플랜은 분당 10회 제한이라, 새로고침·무한 스크롤에서 429가 날 수 있다. 응답을 캐싱하거나 한 번에 `limit`을 크게 받아 쓴다.
- 공개 저장소에 API 키를 커밋하지 않는다.

## 7. 실제 호출 테스트 결과 (2026-11-09, Free 플랜 키)

| 엔드포인트 | 결과 |
|---|---|
| `GET /images/search?limit=2&has_breeds=1&include_breeds=1&order=RANDOM` | 200, 품종 정보 포함 |
| `GET /images/search?limit=1&order=RAND` | 200 (`RAND`, `RANDOM` 둘 다 동작) |
| `GET /breeds?limit=2` | 200 |
| `GET /breeds/search?q=siamese` | 200 |
| `GET /images/KWdLHmOqc` | 200 |
| `GET /categories` | 200, 빈 배열 `[]` |
| `GET /facts/random` | **403** `not available on free plan, upgrade for access` |
| `GET /accounts/quota` | 200 `{"quota":15,"usage":0,"remaining":15}` |
| `GET /accounts/features` | 200, 빈 배열 `[]` |

- Free 플랜에서도 이미지 검색, 품종 목록/검색, 이미지 단건 조회는 정상 동작한다.
- facts는 Free 플랜에서 403이다.
- 카테고리는 비어 있다.
- 이미지 검색의 `breeds` 배열이 비어 있는 사진도 많으므로, 품종 사진만 필요하면 `has_breeds=1`을 붙인다.

## 8. 문서 간 불일치 (사용 전 확인)

1. Base URL: `api.thecatapi.com/v1` (본문) vs `beta-api.thecatapi.com/v1` (OpenAPI 요약)
2. `upload-sync` 호스트: `upload.thecatapi.com` vs `api.thecatapi.com`
3. `order` 값: `RAND` vs `RANDOM`
4. `images/search`의 `mime_types`, `size`, `category_ids` 지원 여부
5. 투표 `value`의 허용 값
