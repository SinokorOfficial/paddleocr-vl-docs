# PaddleOCR-VL 문서 추출

**PaddleOCR-VL-1.6** 모델을 **vLLM**으로 서빙하고, 이미지·PDF를 업로드하면 구조화된 **Markdown/JSON**으로 추출하는 웹 애플리케이션입니다.

<div class="grid cards" markdown>

- :material-rocket-launch: **[실행 (Docker)](setup.md)**

    GPU 서버에서 컨테이너 한 개로 기동

- :material-api: **[API](api.md)**

    `POST /api/extract` 하나로 문서 → Markdown/JSON

- :material-cog: **[환경변수](configuration.md)**

    GPU 배치·동시성·품질 옵션 튜닝

- :material-shield-check: **[품질 보강 파이프라인](quality.md)**

    반복 폭주 · 회전 문서 · 도장 간섭 자동 복구

</div>

## 무엇을 하나

| 입력 | 출력 |
|---|---|
| PNG · JPG · BMP · TIFF · WEBP · **PDF**(다페이지) | 구조화 Markdown (표는 HTML `<table>`, 수식·다국어 텍스트) + 페이지별 JSON |

- 단일 OCR이 아니라 **레이아웃 분석 → 영역별 VLM 인식** 2단계 파이프라인 — 표·수식·읽는 순서를 복원한다
- 기본 설정은 **텍스트 전용 출력** (그림 임베드는 옵션)
- 문제 페이지(반복 폭주·도장 간섭)는 **자동 감지 후 해당 페이지만 재추출**

## 문서 구성

| 문서 | 내용 |
|---|---|
| [실행 (Docker)](setup.md) | 요구사항, 빌드·기동 |
| [환경변수](configuration.md) | 웹앱/vLLM 설정 레퍼런스 |
| [아키텍처](architecture.md) | 2단계 파이프라인, GPU 배치, 요청 처리 모델 |
| [품질 보강 파이프라인](quality.md) | 폭주 3단 방어 · 방향 보정 · 도장 재추출 · 논블로킹 |
| [성능](performance.md) | 실측 수치, CUDA Graph, 튜닝 레버 |
| [API](api.md) | 엔드포인트 명세와 예시 |
| [트러블슈팅](troubleshooting.md) | 증상별 원인·조치 |

## 참고 자료

| 자료 | 링크 |
|---|---|
| 📄 논문 (PaddleOCR-VL, 0.9B VLM) | [arXiv:2510.14528](https://arxiv.org/abs/2510.14528) |
| 🤗 모델 카드 | [PaddlePaddle/PaddleOCR-VL-1.6](https://huggingface.co/PaddlePaddle/PaddleOCR-VL-1.6) |
| :simple-github: PaddleOCR | [github.com/PaddlePaddle/PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) |
| ⚡ vLLM | [github.com/vllm-project/vllm](https://github.com/vllm-project/vllm) |

레퍼런스 하드웨어·운영 환경은 [아키텍처 → 레퍼런스 운영 환경](architecture.md#레퍼런스-운영-환경) 참조.

!!! note "공개판 안내"
    본 문서는 공개판으로, 사내 전용 정보(내부 IP·호스트명·실운영 문서)는 포함하지 않습니다. 코드 저장소는 사내 전용(private)입니다.
