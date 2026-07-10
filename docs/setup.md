# 실행 (Docker)

## 요구사항

| 항목 | 조건 |
|---|---|
| GPU | NVIDIA, VRAM 8GB+ (12GB 권장) |
| 드라이버 | **CUDA 13.x 지원 버전** — 베이스 이미지가 CUDA 13이므로 필수 |
| 호스트 | Docker + NVIDIA Container Toolkit |
| 디스크 | 이미지 ~25GB (모델 가중치 포함 bake) |

!!! warning "드라이버 버전 주의"
    GeForce 계열은 CUDA forward-compatibility를 지원하지 않는다. 컨테이너가 CUDA 13이면 **호스트 드라이버도 CUDA 13.x를 지원**해야 한다 (`nvidia-smi` 우상단 `CUDA Version` 확인). 구형 드라이버(CUDA 12.x)에서는 vLLM이 기동하지 않는다.

!!! info "Windows 호스트"
    vLLM은 Windows 네이티브를 지원하지 않는다 → **WSL2(Ubuntu) 안에서** Docker를 구동한다. WSL 안에서 `nvidia-smi`로 GPU가 보여야 한다.

## 빌드

```bash
git clone <code-repo-url>
cd paddleocr-vl-extractor
docker build -t paddleocr-vl-extractor .
```

- 모델(PaddleOCR-VL-1.6)은 기본적으로 **빌드 시 이미지에 포함**된다(오프라인 동작). 빼려면 `--build-arg BAKE_MODEL=0`
- 최초 빌드는 수십 분 소요 (CUDA 베이스 + vLLM + paddle 설치)

## 기동

```bash
docker run -d --restart unless-stopped --name paddleocr-vl \
  -e LAYOUT_DEVICE=gpu:1 \
  --gpus all --shm-size 2g -p 8000:8000 \
  paddleocr-vl-extractor:latest
```

- GPU가 1장이면 `-e LAYOUT_DEVICE=gpu:0`(기본) 또는 생략
- 최초 기동 시 모델 로딩 **~1–2분** — `GET /api/health` 가 200이면 준비 완료
- 웹 UI: `http://HOST:8000` / API: [API 문서](api.md) 참조

## 상태 확인

```bash
curl http://localhost:8000/api/health
# {"server":"ok","vlm_reachable":true,"pipeline_loaded":true,...}
docker logs -f paddleocr-vl
```

!!! tip "WSL2 상시 운영"
    WSL 배포판은 Windows 쪽 `wsl.exe` 클라이언트 연결이 없으면 **유휴 종료**된다. 배포판 안의 프로세스(tmux·systemd)로는 막을 수 없다. Windows 예약작업으로 `wsl -d <배포판> -- sleep infinity` 홀더를 상시 유지할 것. [트러블슈팅](troubleshooting.md) 참조.
