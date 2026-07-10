# PaddleOCR-VL Document Extractor — 개발 문서

PaddleOCR-VL-1.6 모델을 vLLM으로 서빙하고, 이미지·PDF를 업로드하면 구조화된 Markdown/JSON으로 추출하는 웹 애플리케이션.

---

## 1. 아키텍처

단일 OCR이 아니라 **레이아웃 분석 → 영역별 VLM 인식** 2단계 파이프라인이다.

```
[브라우저/클라이언트]  :8000
    │ 업로드 (이미지/PDF)
    ▼
┌────────────────────────────┐      ┌────────────────────────────┐
│ 웹앱 (FastAPI)      :8000  │ HTTP │ VLM 서버 (vLLM)     :8118  │
│ · 레이아웃 분석 PP-DocLayoutV3 │ ───► │ · PaddleOCR-VL-1.6 서빙    │
│ · 문서방향 감지 PP-LCNet doc_ori│ ◄─── │ · OpenAI 호환 API          │
│ · 결과 Markdown/JSON 조립   │      │ · 텍스트/표/수식/차트 인식  │
└────────────────────────────┘      └────────────────────────────┘
```

- **페이지 간 순차, 페이지 내 영역 병렬** (기본 동시 9영역)
- GPU 2장 구성 시: GPU0 = vLLM(인식), GPU1 = 레이아웃 (`LAYOUT_DEVICE=gpu:1`) 으로 분리 권장
- 추출 요청은 서버에서 **1건씩 직렬 처리** (내부 락). 동시 업로드는 자동으로 큐잉됨

## 2. 실행 (Docker)

```bash
docker build -t paddleocr-vl-extractor .
docker run -d --restart unless-stopped --name paddleocr-vl \
  -e LAYOUT_DEVICE=gpu:1 --gpus all --shm-size 2g -p 8000:8000 \
  paddleocr-vl-extractor:latest
```

- 요구사항: NVIDIA GPU(드라이버 CUDA 13 지원 버전), Docker + NVIDIA Container Toolkit
- 베이스 이미지가 CUDA 13이므로 **호스트 드라이버가 CUDA 13.x를 지원해야** 한다 (GeForce는 forward-compat 미지원)
- Windows 호스트라면 WSL2(Ubuntu) 안에서 구동 (vLLM은 Windows 네이티브 미지원)
- 최초 기동 시 모델 로딩 ~1–2분

## 3. API

### `POST /api/extract`
문서 1건 추출. `multipart/form-data`, 필드명 `file` (png/jpg/jpeg/bmp/tif/tiff/webp/pdf).

```bash
curl -X POST http://HOST:8000/api/extract -F "file=@문서.pdf"
```

응답:

```json
{
  "ok": true,
  "request_id": "a1b2c3d4e5f6",
  "filename": "문서.pdf",
  "num_pages": 2,
  "elapsed_sec": 15.3,
  "retried_pages": [],
  "stamp_cleaned_pages": [1],
  "markdown": "<!-- page:1 -->\n\n## 페이지 1\n\n...",
  "pages": [
    { "page": 1, "markdown": "...", "json": { } },
    { "page": 2, "markdown": "...", "json": { } }
  ]
}
```

| 필드 | 의미 |
|---|---|
| `elapsed_sec` | 순수 서버 처리시간 (큐 대기 제외) |
| `pages[].page` | 페이지 순서 번호 (1..N) |
| `retried_pages` | 반복 폭주가 감지되어 재추출된 페이지 |
| `stamp_cleaned_pages` | 빨간 도장 제거 후 재추출로 교체된 페이지 |
| `markdown` | 전체 병합본. 다페이지면 `<!-- page:N -->` 마커 + `## 페이지 N` 헤더 포함 |

- 표는 HTML `<table>` 로, 수식·다국어 텍스트 포함
- 기본 설정에서 **그림은 출력에 포함되지 않음** (텍스트 전용, `EMBED_IMAGES=1` 로 변경 가능)

### `GET /api/health`
서버·vLLM·파이프라인 상태. `GET /docs` 에 Swagger UI.

## 4. 환경변수

### 웹앱 (컨테이너 실행 시 `-e` 로)

| 변수 | 기본 | 설명 |
|---|---|---|
| `LAYOUT_DEVICE` | `gpu:0` | 레이아웃 모델 장치. GPU 2장이면 `gpu:1` 로 분리 권장, 부족하면 `cpu` |
| `VL_MAX_CONCURRENCY` | `9` | 페이지 내 영역 동시 인식 수 |
| `MAX_PIXELS` | `1310720` | 영역 이미지 픽셀 상한. 고해상도 대형 표 위주면 `786432` 로 ~20% 가속 |
| `MAX_NEW_TOKENS` | `2048` | 영역당 생성 토큰 상한 (폭주 방지) |
| `USE_DOC_ORIENTATION` | `1` | 페이지 방향(90/180/270°) 자동 감지·회전. 회전 스캔 대응 |
| `EMBED_IMAGES` | `0` | `1`이면 그림을 base64 로 markdown 에 임베드 |
| `STAMP_RETRY` | `1` | 빨간 도장 감지 시 도장 제거 재추출 (아래 6.3) |
| `STAMP_RED_RATIO` | `0.002` | 도장 재추출 트리거 임계값 (페이지 내 빨간 픽셀 비율) |

### vLLM 서버 (`scripts/serve_vlm.sh`)

| 변수 | 기본 | 설명 |
|---|---|---|
| `GPU_MEM_UTIL` | `0.55` | vLLM GPU 메모리 점유율. VRAM 여유 시 상향 |
| `MAX_MODEL_LEN` | `8192` | 최대 컨텍스트 |
| `MAX_NUM_SEQS` | `10` | 동시 시퀀스 수 |
| `ENFORCE_EAGER` | `0` | `0`=CUDA Graph(decode-only, ~30% 빠름) / `1`=eager 롤백 |

## 5. 성능

실측 (RTX 3080 Ti, 단독 실행 기준):

| 문서 유형 | 쪽당 처리시간 |
|---|---|
| 일반 텍스트 문서 | ~2초 |
| 슬라이드·그림 포함 | ~3.5–4초 |
| 밀집 표·인보이스 | ~3–4초 |
| 고해상도 대형 표 | ~10초+ |

- **CUDA Graph vs eager** (동일 문서 A/B): eager 5.26s/쪽 → CUDA Graph 3.76s/쪽 (**~29% 개선**)
- CUDA Graph 캡처 배치 = [1,2,4,8]. 동시 영역이 9–10이면 eager 폴백되므로 밀집 문서는 이득이 줄어든다 (필요 시 `VL_MAX_CONCURRENCY=8`)
- 다페이지 문서 총 시간 ≈ 쪽수 × 쪽당 시간 (페이지 순차 처리)
- 여러 문서 동시 업로드 시 직렬 큐잉: 체감 시간 = 대기 + 처리. 순수 처리시간은 `elapsed_sec` 로 확인

## 6. 품질 보강 파이프라인 (커스텀)

원본 파이프라인 위에 추가된 자동 복구 로직. 모두 **문제 페이지에만** 적용되어 정상 문서엔 비용이 없다.

### 6.1 반복 폭주(repetition degeneration) 3단 방어
특정 영역(큰 숫자·괘선·비텍스트 오인)에서 VLM이 같은 토큰(예: `0`)을 토큰 상한까지 반복 출력하는 현상 대응.

1. **예방**: `repetition_penalty=1.15`
2. **페이지 재추출**: 같은 문자 41자+ 연속 감지 시 해당 페이지만 anti-loop 파라미터(temperature 0.3, rep 1.4)로 재추출 → 폭주 없으면 교체 (`retried_pages`)
3. **안전망**: 그래도 남으면 `문자×10 …[반복 N자 축약]` 으로 트림. 40자리 이하 정상 숫자는 보존

### 6.2 문서 방향 자동 보정
`PP-LCNet_x1_0_doc_ori` 분류 모델로 페이지 방향(0/90/180/270°)을 감지해 자동 회전 후 인식. 90도 돌아간 스캔 문서 대응. 단, **페이지 안 일부만 세로인 텍스트**(측면 라벨, 세로 스탬프)는 모델 한계로 미해결.

### 6.3 빨간 도장 자동 재추출
도장이 표 위에 겹치면 VLM이 해당 컬럼 값을 통째로 누락하는 사례가 있다 (실검증: 도장 근처 금액 셀들이 빈 값으로 추출됨).

```
1차 추출 → 페이지 빨간 픽셀 비율 > 임계값?
  → 페이지를 pdftoppm 으로 재렌더 → 빨강→흰색 치환 → 재추출
  → 내용 점수 비교 후 좋아졌으면 교체 (stamp_cleaned_pages)
```

- **내용 점수** = 채워진 표 셀 수×50 + 실질 문자수. 단순 글자수 비교는 도장(그림 마크업) 소실 때문에 오판하므로 셀 충전도를 가중
- 빨간 **글자**가 지워져 내용이 줄면 원본 유지 (안전)
- 렌더러 주의: pypdfium2 는 도장 안티앨리어싱을 다르게 그려 빨강 제거가 덜 먹힌다 → **pdftoppm(poppler) 우선**, pypdfium2 폴백

### 6.4 논블로킹 처리
추출은 threadpool 에서 실행되고 파이프라인은 락으로 직렬화된다. **문서 처리 중에도** `/api/health`·정적 페이지는 즉시 응답한다 (수정 전에는 추출 동안 서버 전체가 블로킹됐음).

## 7. 트러블슈팅

| 증상 | 원인/조치 |
|---|---|
| 웹은 뜨는데 추출 시 503 | vLLM 로딩 중(기동 후 ~1–2분) 또는 미기동. 컨테이너 로그 확인 |
| CUDA out of memory | `GPU_MEM_UTIL` 하향(0.5/0.45) 또는 `LAYOUT_DEVICE=cpu` |
| 기동 직후 첫 요청이 유난히 느림 | 파이프라인 지연 초기화(1회성, +30초 내외). 정상 |
| 같은 문자 무한 반복 출력 | 6.1 로 자동 처리됨. 그래도 남으면 `MAX_NEW_TOKENS` 하향 검토 |
| 표 셀 값 누락 (도장 문서) | 6.3 이 자동 처리. `stamp_cleaned_pages` 확인. 임계값은 `STAMP_RED_RATIO` |
| 회전 스캔 인식 불량 | `USE_DOC_ORIENTATION=1`(기본) 확인 |
| 처리가 비정상적으로 오래 걸림 | 동시 업로드 큐잉 여부 확인 (한 번에 1건 처리). `elapsed_sec` 과 체감 시간 비교 |
| WSL2 운영 시 서비스가 밤사이 죽음 | WSL 배포판은 Windows 쪽 wsl.exe 클라이언트가 없으면 유휴 종료됨. 배포판 안 프로세스(tmux/systemd)로는 못 막는다 → Windows 예약작업으로 `wsl -d <배포판> -- sleep infinity` 홀더를 상시 유지 |

## 8. 저장소 구성

```
├─ app/server.py        # FastAPI 백엔드 (레이아웃 + VLM 원격 파이프라인, 품질 보강 로직)
├─ static/index.html    # 프론트엔드 (업로드·미리보기)
├─ scripts/
│  ├─ serve_vlm.sh      # vLLM 서버 실행 (컨테이너 진입점에서 사용)
│  └─ run_app.sh        # 웹앱 실행 (컨테이너 진입점에서 사용)
├─ docker/entrypoint.sh # vLLM 기동 대기 후 웹앱 실행
├─ Dockerfile           # CUDA13 베이스, venv 2개(vLLM/앱), 모델 bake
├─ docker-compose.yml
├─ samples/             # 테스트 샘플
└─ docs/                # 본 문서
```
