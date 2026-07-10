# paddleocr-vl-docs

**PaddleOCR-VL 문서 추출 서비스**의 개발·운영 문서 (공개판).

> 📖 **문서 사이트**: https://sinokorofficial.github.io/paddleocr-vl-docs/

PaddleOCR-VL-1.6(0.9B VLM)을 vLLM으로 서빙하고 이미지·PDF를 구조화된 Markdown/JSON으로 추출하는 서비스의 아키텍처 · 실행 방법 · API · 성능 · 품질 보강 파이프라인을 다룹니다. 코드 저장소는 사내 전용(private)입니다.

## 구성

```
├─ mkdocs.yml                  # MkDocs Material 설정 (한국어, 조직 문서 표준)
├─ requirements.txt            # mkdocs / mkdocs-material / pymdown-extensions
├─ .github/workflows/docs.yml  # main 푸시 시 GitHub Pages 자동 빌드·배포
└─ docs/
   ├─ index.md                 # 홈
   ├─ setup.md                 # 실행 (Docker)
   ├─ configuration.md         # 환경변수 레퍼런스
   ├─ architecture.md          # 아키텍처 · 모델 출처 · 레퍼런스 하드웨어
   ├─ quality.md               # 품질 보강 (폭주 방어 · 방향 보정 · 도장 재추출)
   ├─ performance.md           # 성능 실측 · 튜닝
   ├─ api.md                   # API 명세
   └─ troubleshooting.md       # 트러블슈팅
```

## 문서 수정 방법

1. `docs/*.md` 수정 (페이지 추가 시 `mkdocs.yml` 의 `nav` 에도 등록)
2. `main` 에 푸시 → GitHub Actions 가 자동 빌드·배포 (1–2분)

로컬 미리보기:

```bash
pip install -r requirements.txt
mkdocs serve   # http://127.0.0.1:8000
```

## 원칙

- **사내 전용 정보 금지** — 내부 IP·호스트명·계정·실운영 문서를 본 공개 리포에 올리지 않는다
- 문서는 한국어, 조직 문서 표준(MkDocs Material) 을 따른다

---
관리: 장금상선 AI 파트 (it-team)
