# 트러블슈팅

## 서비스

| 증상 | 원인 / 조치 |
|---|---|
| 웹은 뜨는데 추출 시 `503` | vLLM 로딩 중(기동 후 ~1–2분) 또는 미기동. `docker logs` 확인 |
| `CUDA out of memory` | `GPU_MEM_UTIL` 하향(0.5/0.45) 또는 `LAYOUT_DEVICE=cpu` |
| vLLM이 기동 자체를 못 함 | 호스트 드라이버가 CUDA 13 미지원 (`nvidia-smi` 로 CUDA Version 확인 → 드라이버 업그레이드) |
| 기동 직후 첫 요청이 유난히 느림 | 파이프라인 지연 초기화 (1회성 +30초 내외). 정상 |
| 처리가 비정상적으로 오래 걸림 | 동시 업로드 큐잉 (1건씩 직렬 처리). `elapsed_sec` 과 체감 시간 비교 — 차이가 크면 대기가 원인 |

## 추출 품질

| 증상 | 원인 / 조치 |
|---|---|
| 같은 문자(예: `0`)가 무한 반복 | [폭주 3단 방어](quality.md#반복-폭주-3단-방어)가 자동 처리. `retried_pages` 확인. 잦으면 `MAX_NEW_TOKENS` 하향 검토 |
| 도장 겹친 표의 값 누락 | [도장 재추출](quality.md#빨간-도장-자동-재추출)이 자동 처리. `stamp_cleaned_pages` 확인. 감지 안 되면 `STAMP_RED_RATIO` 하향 |
| 회전 스캔 인식 불량 | `USE_DOC_ORIENTATION=1`(기본) 확인. 페이지 **일부만** 세로인 텍스트는 모델 한계 |
| 아주 작은 글자 뭉개짐 | `MAX_PIXELS` 상향 (기본 1310720 → 2073600). 속도와 트레이드오프 |
| 그림이 결과에 없음 | 의도된 기본값 (텍스트 전용). `EMBED_IMAGES=1` 로 변경 |

## WSL2 운영 (Windows 호스트)

!!! danger "배포판 유휴 종료 — 가장 흔한 함정"
    WSL2 배포판의 수명은 **배포판 안 프로세스가 아니라 Windows 쪽 `wsl.exe` 클라이언트 연결**이 결정한다. 마지막 클라이언트가 끊기면 tmux·systemd·docker 가 돌고 있어도 배포판이 통째로 종료된다 → 서비스가 "밤사이 죽는" 패턴.

    **해결**: Windows 예약작업으로 홀더를 상시 유지 —

    ```
    schtasks /create /tn WSL_KeepAlive /tr C:\ops\wsl_keepalive.bat ^
      /sc minute /mo 5 /ru <계정> /rp <비밀번호> /rl HIGHEST /f
    ```

    `wsl_keepalive.bat` 내용: `"C:\Program Files\WSL\wsl.exe" -d <배포판> -u root -- sleep infinity`
    (5분 주기 + 중복 실행 무시 정책 → 홀더 1개 상시 유지, 죽어도 5분 내 부활)

| 증상 | 원인 / 조치 |
|---|---|
| 재부팅/방치 후 접속 불가 | 위 유휴 종료. keep-alive 예약작업 등록 확인 |
| WSL 안에서 `nvidia-smi` 없음 | Windows NVIDIA 드라이버가 WSL 활성화 **전에** 설치된 경우 WSL용 라이브러리 미배포 → 드라이버 재설치(또는 pnputil 로 INF 설치) 후 `wsl --shutdown` |
| 포트포워딩 끊김 | WSL IP 는 배포판 재시작마다 변경됨 → `netsh portproxy` 의 connectaddress 갱신 필요 |
| Windows 쪽 포트 충돌 | `Get-NetTCPConnection -LocalPort <port>` 로 선점 프로세스 확인 후 다른 포트로 포워딩 |

## 진단 커맨드 모음

```bash
# 컨테이너/앱 상태
docker ps && docker logs --tail 50 paddleocr-vl
curl -s http://localhost:8000/api/health

# GPU 사용 현황 (1초 갱신)
nvidia-smi -l 1

# vLLM 상세 로그 (컨테이너 내부 파일)
docker exec paddleocr-vl tail -50 /opt/ocr/vlm_serve.log
```
