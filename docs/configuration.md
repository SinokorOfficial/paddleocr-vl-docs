# 환경변수

모두 `docker run -e KEY=VALUE` 로 지정한다.

## 웹앱 (추출 파이프라인)

| 변수 | 기본 | 설명 |
|---|---|---|
| `LAYOUT_DEVICE` | `gpu:0` | 레이아웃 모델 장치. GPU 2장이면 `gpu:1` 로 vLLM(gpu:0)과 분리 권장. 메모리 부족 시 `cpu` |
| `VL_MAX_CONCURRENCY` | `9` | 페이지 내 영역 동시 인식 수. CUDA Graph 캡처 배치(≤8)에 맞추려면 `8` |
| `MAX_PIXELS` | `1310720` | 영역 이미지 픽셀 상한. 고해상도 대형 표 위주 문서는 `786432` 로 ~20% 가속 |
| `MAX_NEW_TOKENS` | `2048` | 영역당 생성 토큰 상한 (반복 폭주 시 피해 한도) |
| `PADDLE_PDX_PDF_RENDER_SCALE` | `2.8` | **PDF 내부 렌더 스케일**(≈200dpi). paddlex 기본 2.0(144dpi)은 선이 가늘고 밀집된 표에서 구조 인식 붕괴(셀 값 누락)를 유발 — 상향 필수 |
| `USE_DOC_ORIENTATION` | `1` | 페이지 방향(90/180/270°) 자동 감지·회전 (PP-LCNet doc_ori) |
| `EMBED_IMAGES` | `0` | `1`이면 그림을 base64 로 markdown 에 임베드. 기본은 텍스트 전용 |
| `STAMP_RETRY` | `1` | 빨간 도장 감지 시 도장 제거 재추출 ([품질 보강](quality.md#빨간-도장-자동-재추출) 참조) |
| `STAMP_RED_RATIO` | `0.002` | 도장 재추출 트리거 — 페이지 내 빨간 픽셀 비율 임계값 |
| `VLM_SERVER_URL` | `http://127.0.0.1:8118` | VLM 서버 주소 (컨테이너 내부) |
| `PORT` | `8000` | 웹앱 포트 |

## vLLM 서버 (`scripts/serve_vlm.sh`)

| 변수 | 기본 | 설명 |
|---|---|---|
| `GPU_MEM_UTIL` | `0.55` | vLLM GPU 메모리 점유율. VRAM 여유 시 상향하면 KV 캐시·동시성↑ |
| `MAX_MODEL_LEN` | `8192` | 최대 컨텍스트 (OCR 영역은 짧아 충분) |
| `MAX_NUM_SEQS` | `10` | 서버 동시 시퀀스 수 |
| `ENFORCE_EAGER` | `0` | `0` = CUDA Graph(decode-only, **~30% 빠름**) / `1` = eager 롤백 |

## 권장 프리셋

=== "GPU 2장 (12GB×2)"

    ```bash
    -e LAYOUT_DEVICE=gpu:1
    # vLLM은 GPU0, 레이아웃은 GPU1 — 상호 간섭 없음
    ```

=== "GPU 1장 (8GB)"

    ```bash
    -e LAYOUT_DEVICE=cpu -e GPU_MEM_UTIL=0.5
    # 레이아웃을 CPU로 내려 vLLM에 VRAM 양보
    ```

=== "고해상도 스캔 위주"

    ```bash
    -e MAX_PIXELS=786432
    # 전폭 대형 표에서 비전 인코딩 비용 절감 (~20% 가속, 품질 거의 유지)
    ```
