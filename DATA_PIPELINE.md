# 데이터 파이프라인 문서

> **프로젝트**: 멀티 에이전트 기반 액체화물 하역 스케줄링 및 안전 관제 시스템  
> **작성일**: 2026-06-24  
> **대상**: 팀원 / 멘토

---

## 목차

1. [전체 구조](#1-전체-구조)
2. [데이터 흐름](#2-데이터-흐름)
3. [수집 데이터 4종](#3-수집-데이터-4종)
4. [디렉토리 구조](#4-디렉토리-구조)
5. [실행 방법](#5-실행-방법)
6. [Staging 스키마](#6-staging-스키마)
7. [공통 전처리 규칙](#7-공통-전처리-규칙)
8. [현재 진행 상황](#8-현재-진행-상황)
9. [환경 설정](#9-환경-설정)

---

## 1. 전체 구조

```
공공 API (4종)
    │
    ▼
[Collector]  ──→  data/raw/       (JSON, 원본 그대로 보존)
    │
    ▼
[Preprocessor] ──→  data/staging/  (CSV, 표준화·정제 완료)
    │
    ▼
[Multi-Agent System]  (분석 / 스케줄링 / 안전 관제)
```

수집기(Collector)는 API를 호출해 원본 JSON을 저장하고,  
전처리기(Preprocessor)는 이를 읽어 표준 컬럼·UTC 시각·품질 플래그를 적용한 CSV로 변환합니다.

---

## 2. 데이터 흐름

```
API 호출
  ↓
data/raw/<도메인>/<파일명>_YYYYMMDD_raw.json   ← 원본 보존 (절대 수정 X)
  ↓
data/staging/<파일명>_stg.csv                  ← 정제·표준화 완료
```

- raw 파일은 날짜별로 누적 저장 (같은 날 여러 번 실행 시 중복 제거 후 append)
- staging 파일은 항상 전체 재생성 (raw 전체를 읽어 덮어씀)

---

## 3. 수집 데이터 4종

### 3-1. 조위 (Tide) — `tide_collector.py`

| 항목 | 내용 |
|---|---|
| API | 공공데이터포탈 국립해양조사원 `GetDTRecentApiService` |
| URL | `https://apis.data.go.kr/1192136/dtRecent/GetDTRecentApiService` |
| 관측소 | 울산 조위관측소 `DT_0020` |
| 수집 주기 | 1시간 간격 (`min=60`) |
| 수집 항목 | 조위, 풍향·풍속·돌풍, 기온, 기압, 수온, 염분, 유향·유속 |

**주요 API 파라미터**

| 파라미터 | 값 | 설명 |
|---|---|---|
| `obsCode` | `DT_0020` | 울산 관측소 |
| `min` | `60` | 1시간 간격 |
| `numOfRows` | `300` | 최대 조회 건수 |
| `reqDate` | `YYYYMMDD` | 요청 날짜 (생략 시 당일) |

---

### 3-2. 파고 (Wave) — `wave_collector.py`

| 항목 | 내용 |
|---|---|
| API | 기상청 APIHUB KMA BUOY |
| URL | `https://apihub.kma.go.kr/api/typ01/url/kma_buoy.php` |
| 관측소 | 울산 부이 `STN 22189` |
| 수집 주기 | 10분 간격 (API 기본) |
| 수집 항목 | 유의파고·최대파고·평균파고, 파주기, 파향, 풍향·풍속(듀얼센서)·돌풍, 기온, 해수온도, 기압, 습도 |

**응답 형식**: 공백 구분 텍스트 (JSON 아님) — 컬럼 순서 고정

---

### 3-3. 항만기상 (Weather) — `weather_collector.py`

| 항목 | 내용 |
|---|---|
| API | 해양수산부 항만기상정보시스템 |
| URL | `http://marineweather.nmpnt.go.kr:8001/openWeatherNow.do` |
| 관측소 | 울산항동방파제서단등대 `MMSI 1041519` |
| 수집 주기 | 실시간 (호출 시 최신값 1건) |
| 수집 항목 | 풍향, 풍속, 기온, 습도, 기압, 시정 |

---

### 3-4. MSDS 화학물질 안전보건자료 — `msds_api_collector.py`

| 항목 | 내용 |
|---|---|
| API | 공공데이터포탈 안전보건공단 MSDS |
| URL | `https://apis.data.go.kr/B552468/msdschem/` |
| 수집 대상 | 울산항 취급 화학물질 **20종** |
| 조회 방식 | CAS번호 정확 검색 (`searchCnd=1`) → chemId 확보 → 16개 섹션 상세 조회 |
| 수집 항목 | 인화점, 끓는점, 증기압, 비중, UN번호, IMDG 등급, 용기등급, EMS코드, GHS 신호어, 유해성 분류 |

**수집 화학물질 목록 (20종)**

| 분류 | 화학물질 | CAS번호 |
|---|---|---|
| 정유제품 | 나프타 | 64741-41-9 |
| 정유제품 | 가솔린 | 8006-61-9 |
| 정유제품 | 경유 | 64742-86-5 |
| 정유제품 | 등유 | 8020-83-5 |
| BTX 방향족 | 벤젠 | 71-43-2 |
| BTX 방향족 | 톨루엔 | 108-88-3 |
| BTX 방향족 | 자일렌 | 1330-20-7 |
| BTX 방향족 | 스티렌 | 100-42-5 |
| 알코올류 | 메탄올 | 67-56-1 |
| 알코올류 | 에탄올 | 64-17-5 |
| 알코올류 | 이소프로판올 | 67-63-0 |
| 지방족/케톤 | 헥산 | 110-54-3 |
| 지방족/케톤 | 사이클로헥산 | 110-82-7 |
| 지방족/케톤 | 아세톤 | 67-64-1 |
| 지방족/케톤 | 메틸에틸케톤 | 78-93-3 |
| LPG/에폭사이드 | 프로판 | 74-98-6 |
| LPG/에폭사이드 | 부탄 | 106-97-8 |
| LPG/에폭사이드 | 프로필렌옥사이드 | 75-56-9 |
| 무기화학 | 황산 | 7664-93-9 |
| 무기화학 | 암모니아 | 7664-41-7 |

**CAS번호로 조회하는 이유**: 화학물질명으로 검색하면 유사 물질 수십 건이 반환됨. CAS번호 정확 검색(`searchCnd=1`)으로 1건만 조회하여 오수집 방지.

---

## 4. 디렉토리 구조

```
smart-port-multi-agent/
├── data_pipeline/
│   ├── common_preprocessing.py       # 공통 전처리 유틸 (정규화·메타·검증)
│   ├── collectors/
│   │   ├── tide_collector.py         # 조위 수집
│   │   ├── wave_collector.py         # 파고 수집
│   │   ├── weather_collector.py      # 항만기상 수집
│   │   └── msds_api_collector.py     # MSDS 화학물질 수집
│   └── preprocessors/
│       ├── tide_preprocessor.py      # 조위 전처리
│       ├── wave_preprocessor.py      # 파고 전처리
│       ├── weather_preprocessor.py   # 항만기상 전처리
│       └── msds_preprocessor.py      # MSDS 전처리
│
├── data/
│   ├── raw/                          # 원본 JSON (수정 금지)
│   │   ├── tide/    tide_obs_YYYYMMDD_raw.json
│   │   ├── wave/    wave_obs_YYYYMMDD_raw.json
│   │   ├── weather/ weather_obs_YYYYMMDD_raw.json
│   │   └── msds/    msds_chemical_YYYYMMDD_raw.json
│   └── staging/                      # 정제된 CSV
│       ├── tide_obs_stg.csv
│       ├── wave_obs_stg.csv
│       ├── weather_obs_stg.csv
│       └── msds_chemical_stg.csv
│
├── .env                              # API 키 (Git 제외)
├── .env.example                      # API 키 템플릿
└── DATA_PIPELINE.md                  # 본 문서
```

---

## 5. 실행 방법

프로젝트 루트(`smart-port-multi-agent/`)에서 실행합니다.

### 수집 (Collector)

```bash
# 조위
python -m data_pipeline.collectors.tide_collector

# 파고
python -m data_pipeline.collectors.wave_collector

# 항만기상
python -m data_pipeline.collectors.weather_collector

# MSDS (약 3분 소요 — 20종 × 16섹션 API 호출)
python -m data_pipeline.collectors.msds_api_collector
```

### 전처리 (Preprocessor)

```bash
# 조위
python -m data_pipeline.preprocessors.tide_preprocessor

# 파고
python -m data_pipeline.preprocessors.wave_preprocessor

# 항만기상
python -m data_pipeline.preprocessors.weather_preprocessor

# MSDS
python -m data_pipeline.preprocessors.msds_preprocessor
```

---

## 6. Staging 스키마

### 6-1. `tide_obs_stg.csv` — 조위

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `station_id` | str | 관측소 코드 (DT_0020) |
| `station_name` | str | 관측소명 (울산) |
| `observed_at_utc` | datetime | 관측 시각 (UTC) |
| `tide_level_cm` | float | 조위 — 기본수준면 기준 (cm) |
| `wind_dir_deg` | float | 풍향 (deg) |
| `wind_speed_ms` | float | 풍속 (m/s) |
| `gust_ms` | float | 최대순간풍속 (m/s) |
| `air_temp_c` | float | 기온 (℃) |
| `air_pressure_hpa` | float | 기압 (hPa) |
| `sea_temp_c` | float | 수온 (℃) |
| `salinity_psu` | float | 염분 (psu) |
| `current_dir_deg` | float | 유향 (deg) |
| `current_speed_cms` | float | 유속 (cm/s) |
| `quality_flag` | str | OK / INVALID_* / MISSING_KEY |
| `source_system` | str | KDPA_KHOA_dtRecent |

---

### 6-2. `wave_obs_stg.csv` — 파고

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `station_id` | str | 부이 지점 번호 (22189) |
| `station_name` | str | 지점명 (울산) |
| `observed_at_utc` | datetime | 관측 시각 (UTC) |
| `wave_height_sig_m` | float | 유의파고 (m) |
| `wave_height_max_m` | float | 최대파고 (m) |
| `wave_height_avg_m` | float | 평균파고 (m) |
| `wave_period_s` | float | 파주기 (s) |
| `wave_dir_deg` | float | 파향 (deg) |
| `wind_dir1_deg` | float | 풍향 센서1 (deg) |
| `wind_speed1_ms` | float | 풍속 센서1 (m/s) |
| `gust1_ms` | float | 돌풍 센서1 (m/s) |
| `wind_dir2_deg` | float | 풍향 센서2 (deg) |
| `wind_speed2_ms` | float | 풍속 센서2 (m/s) |
| `gust2_ms` | float | 돌풍 센서2 (m/s) |
| `air_temp_c` | float | 기온 (℃) |
| `sea_temp_c` | float | 해수온도 (℃) |
| `air_pressure_hpa` | float | 기압 (hPa) |
| `humidity_pct` | float | 습도 (%) |
| `quality_flag` | str | OK / INVALID_* / MISSING_KEY |
| `source_system` | str | KMA_APIHUB_BUOY |

---

### 6-3. `weather_obs_stg.csv` — 항만기상

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `station_id` | str | MMSI 코드 (1041519) |
| `station_name` | str | 관측소명 (울산항동방파제서단등대) |
| `observed_at_utc` | datetime | 관측 시각 (UTC) |
| `wind_dir_deg` | float | 풍향 (deg) |
| `wind_speed_ms` | float | 풍속 (m/s) |
| `air_temp_c` | float | 기온 (℃) |
| `humidity_pct` | float | 습도 (%) |
| `air_pressure_hpa` | float | 기압 (hPa) |
| `visibility_m` | float | 시정 (m) |
| `quality_flag` | str | OK / INVALID_* / MISSING_KEY |
| `source_system` | str | MMPNT_MARINEWEATHER |

---

### 6-4. `msds_chemical_stg.csv` — MSDS 화학물질

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `chem_id` | str | KOSHA 화학물질 ID |
| `cargo_name_raw` | str | KOSHA 등록 한국어 화학물질명 |
| `cargo_name_en` | str | 영문명 |
| `cas_no` | str | CAS 등록번호 |
| `flash_point_celsius` | float | 인화점 (℃, 파싱된 수치) |
| `flash_point_text` | str | 인화점 원문 (출처 포함) |
| `boiling_point_text` | str | 끓는점 원문 |
| `vapor_pressure_text` | str | 증기압 원문 |
| `specific_gravity_text` | str | 비중 원문 |
| `dg_un_no` | str | UN 번호 (위험물 운송 코드) |
| `imdg_class` | str | IMDG 위험성 등급 |
| `packing_group` | str | 용기등급 (Ⅰ/Ⅱ/Ⅲ) |
| `ems_fire` | str | 화재 시 비상조치 EMS 코드 |
| `ems_spill` | str | 유출 시 비상조치 EMS 코드 |
| `ghs_hazard` | str | GHS 유해성·위험성 분류 |
| `signal_word` | str | GHS 신호어 (위험/경고) |
| `h_statements` | str | 유해·위험문구 (H-코드) |
| `emergency_response` | str | 응급조치요령 |
| `fire_response` | str | 폭발·화재 대처 |
| `spill_response` | str | 누출사고 대처 |
| `quality_flag` | str | OK / MISSING_KEY |
| `source_system` | str | KOSHA_MSDS_API |

---

## 7. 공통 전처리 규칙

`common_preprocessing.py`에 정의된 공통 함수를 모든 전처리기가 사용합니다.

| 함수 | 설명 |
|---|---|
| `normalize_nulls()` | `""`, `"null"`, `"-"`, `"N/A"` 등 → `NaN` 통일 |
| `to_numeric_safe()` | 문자열 숫자 변환, 실패 시 NaN (에러 없음) |
| `parse_datetime_kst_to_utc()` | KST → UTC 변환 |
| `add_common_metadata()` | `source_system`, `source_table`, `collected_at_utc`, `quality_flag`, `is_synthetic` 컬럼 추가 |
| `flag_missing_key()` | 기본키 컬럼이 NaN이면 `quality_flag = "MISSING_KEY"` |
| `flag_ulsan_bbox()` | 좌표가 울산 바운딩박스 밖이면 `"OUT_OF_ULSAN_BBOX"` |
| `save_staging_csv()` | `utf-8-sig` 인코딩으로 CSV 저장 (Excel 한글 깨짐 방지) |

**울산 바운딩박스**: 위도 35.30~35.58, 경도 129.18~129.52

---

## 8. 현재 진행 상황

### 완료된 작업

| # | 작업 | 상태 |
|---|---|---|
| 1 | 조위 수집기 구현 (KHOA `GetDTRecentApiService`) | ✅ 완료 |
| 2 | 조위 전처리기 구현 | ✅ 완료 |
| 3 | 파고 수집기 구현 (기상청 APIHUB BUOY STN 22189) | ✅ 완료 |
| 4 | 파고 전처리기 구현 | ✅ 완료 |
| 5 | 항만기상 수집기 구현 (해양수산부 marineweather) | ✅ 완료 |
| 6 | 항만기상 전처리기 구현 | ✅ 완료 |
| 7 | MSDS 수집기 구현 (KOSHA, CAS번호 기준 20종) | ✅ 완료 |
| 8 | MSDS 전처리기 구현 (msdsItemCode 기반 파싱) | ✅ 완료 |
| 9 | 공통 전처리 모듈 구현 | ✅ 완료 |
| 10 | 전체 staging CSV 생성 검증 (quality_flag 100% OK) | ✅ 완료 |

### 미완료 / 제한 사항

| 항목 | 내용 | 이유 |
|---|---|---|
| 강수 데이터 | 수집 불가 | 기상청 `VilageFcstInfoService2` 별도 활용신청 필요 |
| 기상 예보 데이터 | 수집 불가 | 동일 — `getUltraSrtFcst`, `getVilageFcst` 별도 신청 필요 |
| 조위 예보 (고·저조) | 미구현 | 기존 `GetTideFcstHghLwApiService` 제거, 필요 시 재추가 가능 |
| 유향·유속 | 관측값 없음 | 울산 DT_0020 관측소에 해당 센서 미설치 |

---

## 9. 환경 설정

`.env` 파일을 프로젝트 루트에 생성합니다 (`.env.example` 참고).

```env
# 공공데이터포탈 API 키 (디코딩된 값)
KOSHA_MSDS_API_KEY=<발급받은_키>

# 기상청 APIHUB 부이 관측 API 인증키
KMA_BUOY_AUTH_KEY=<발급받은_키>

# 기상청 공공데이터포탈 API 키 (강수·예보용 — 활용신청 후 입력)
KMA_API_KEY=

# 국립해양조사원 API 키
KHOA_API_KEY=<발급받은_키>
```

**패키지 설치**

```bash
pip install -r data_pipeline/requirements.txt
```

---

## 참고 — API 출처 정리

| 데이터 | 기관 | 포탈 |
|---|---|---|
| 조위 | 국립해양조사원 (KHOA) | 공공데이터포탈 data.go.kr |
| 파고 | 기상청 (KMA) | 기상청 APIHUB apihub.kma.go.kr |
| 항만기상 | 해양수산부 (MMAF) | 항만기상정보시스템 marineweather.nmpnt.go.kr |
| MSDS | 안전보건공단 (KOSHA) | 공공데이터포탈 data.go.kr |
