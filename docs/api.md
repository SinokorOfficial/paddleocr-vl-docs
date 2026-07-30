# API

FastAPI 기반. 대화형 문서(Swagger UI)는 서비스의 `/docs` 경로에서 제공된다.

## `POST /api/extract`

문서 1건 추출. `multipart/form-data`, 필드명 **`file`**.

- 지원 형식: `png` `jpg` `jpeg` `bmp` `tif` `tiff` `webp` `pdf`(다페이지)
- 엔진 선택: 폼 필드 `model` = `paddle`(기본) | `ovis` — [엔진 비교](architecture.md) 참조
- 처리 시간: 쪽당 ~2–10초 ([성능](performance.md)) — 클라이언트 타임아웃을 넉넉히 (다페이지 PDF는 분 단위)

=== "curl"

    ```bash
    curl -X POST http://HOST:8000/api/extract -F "file=@document.pdf"
    ```

=== "Python"

    ```python
    import requests

    r = requests.post(
        "http://HOST:8000/api/extract",
        files={"file": open("document.pdf", "rb")},
        timeout=600,
    )
    d = r.json()
    print(d["num_pages"], "pages in", d["elapsed_sec"], "sec")
    for p in d["pages"]:
        print(f"--- page {p['page']} ---")
        print(p["markdown"][:200])
    ```

### 응답

```json
{
  "ok": true,
  "request_id": "a1b2c3d4e5f6",
  "filename": "document.pdf",
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
| `engine` | 처리 엔진 (`paddle` / `ovis`) |
| `pages[].page` | 페이지 순서 번호 (1..N) |
| `pages[].markdown` | 페이지별 추출 결과 (표 = HTML `<table>`) |
| `pages[].json` | 파이프라인 원시 결과 (영역 좌표 등) |
| `retried_pages` | 반복 폭주로 재추출된 페이지 번호 목록 |
| `stamp_cleaned_pages` | 도장 제거 재추출로 교체된 페이지 번호 목록 |
| `markdown` | 전체 병합본 — 다페이지면 `<!-- page:N -->` 마커 + `## 페이지 N` 헤더 포함 |

!!! tip "품질 신호"
    `retried_pages` / `stamp_cleaned_pages` 가 비어있지 않으면 해당 페이지는 자동 복구를 거쳤다는 뜻 — 중요 문서라면 그 페이지를 우선 검수하는 지표로 활용할 수 있다.

### 오류

| 코드 | 의미 |
|---|---|
| `400` | 지원하지 않는 파일 형식 |
| `503` | 파이프라인 초기화 실패 (VLM 서버 미기동 — 로딩 중일 수 있음) |
| `500` | 추출 실패 (상세 메시지 포함) |

## `POST /api/extract_async` · `GET /api/jobs/{job_id}`

**접수번호 방식 (장문서 권장).** 동기 API 는 응답까지 수 분간 데이터가 흐르지 않는 긴 연결이라
중간 프록시가 유휴로 오인해 끊으면 완료된 결과가 유실될 수 있다. 비동기 방식은 짧은 요청만 오가므로 안전하다.

```bash
# 1) 접수 — 즉시 반환
curl -X POST http://HOST:8000/api/extract_async -F "file=@document.pdf" -F "model=ovis"
# → {"job_id": "a1b2c3d4e5f6", "status": "processing", "engine": "ovis"}

# 2) 조회 — 완료 시 result 에 동기 API 와 동일한 전체 결과
curl http://HOST:8000/api/jobs/a1b2c3d4e5f6
```

| 상태 | 의미 |
|---|---|
| `processing` | 대기 또는 처리 중 (`elapsed_sec` 로 경과 확인) |
| `done` | 완료 — `result` 필드에 전체 결과 |
| `error` | 실패 — `error` 필드에 사유 |

- 결과는 완료 후 **1시간 보관** — 클라이언트가 끊겨도 재조회 가능
- 웹 UI 는 이 방식을 사용한다 (1.5초 간격 폴링)

## `GET /api/health`

```json
{
  "server": "ok",
  "vlm_server_url": "http://127.0.0.1:8118",
  "vlm_reachable": true,
  "vlm_model": "PaddleOCR-VL",
  "pipeline_loaded": true,
  "pipeline_error": null,
  "layout_device": "gpu:1"
}
```

- `vlm_reachable=true` + `pipeline_loaded=true` 면 추출 가능 상태
- 기동 직후에는 `pipeline_loaded=false` 일 수 있다 (첫 추출 시 지연 로딩)

## 운영 메모

- **인증 없음** — 내부망 전용을 전제로 한다. 외부 노출 시 별도 게이트웨이/인증 필수
- 결과의 그림 임베드는 기본 비활성 (텍스트 전용). `EMBED_IMAGES=1` 시 `markdown` 에 base64 포함되어 응답이 커짐
- VLM 서버(:8118)는 컨테이너 내부 전용 — 외부 노출 불필요
