# paddle-ocr 서빙하려고 만든 레포

# TODO
## fast api로 테스트 — 완료 (로컬 CPU 검증)
## Genos에서 사용 가능하게 스크립트 작성 — 완료 (detr 구조 이식)

# 진행 로그

## fast api 테스트 (완료)
- 로컬(macOS arm64, CPU)에 `paddlepaddle` + `paddlex[serving,ocr]` 설치.
- `paddlex --serve --pipeline OCR --host 0.0.0.0 --port 8081` 로 기동.
  - 첫 기동 시 서브모델(문서방향분류/UVDoc 언워핑/텍스트라인방향/PP-OCRv6 검출·인식) 자동 다운로드.
  - `/health`, `/docs`, `/openapi.json` 이 실제로 뜸 → PaddleX serve는 내부적으로 FastAPI 기반.
  - OpenAPI 상 엔드포인트: `/health`, `/ocr` (POST, `{"file": <base64>, "fileType": 1}`).
- `preprocessor` 프로젝트의 샘플 PDF(`Information Theory.pdf`) 첫 페이지를 PyMuPDF로 150dpi 렌더링해서
  `/ocr`에 전송 → 200 OK, 17초, 38개 텍스트 정확히 인식 (원문과 거의 동일한 텍스트 복원 확인).
  - 주의: 첫 요청 시 시각화용 폰트(`simfang.ttf`, ~10MB)를 bcebos.com에서 추가로 받아오느라
    첫 호출은 오래 걸림(60초 타임아웃 걸렸었음) → 실제 서빙 시 워밍업 요청을 미리 보내두는 게 좋음.
- 참고용 레퍼런스: `genonai/doc_parser` 레포의 `genon/serving/paddle/`
  (Dockerfile, config/ocr.yaml — 한국어 인식 모델(`korean_PP-OCRv5_mobile_rec`) 사용,
  config/supervisord.conf, etc/health_checker.sh, etc/smoke_test*.sh, k8s-manifest/, pyproject.toml)와
  `build-script/paddle-ocr-build.config` + `.sh`.
  - `pyproject.toml`은 `paddlepaddle-gpu==3.2.0` + `paddlex==3.3.6` + `paddlex[serving,ocr]` 조합.
  - `config/ocr.yaml`은 pipeline_name: OCR 기반이지만 TextRecognition 모델을 한국어용으로 교체하고
    model_dir을 `/models/paddleocr_model/...`로 지정 (모델 zip을 이미지 빌드 시 구워둠).

## Genos 서빙 스크립트 (완료)
- `detr` 레포 구조(`docker/Dockerfile`, `config/supervisord.conf`, `etc/health_checker.sh`,
  `etc/smoke_test.sh`, `build-script/*`, `k8s-manifest/*`)를 그대로 이식.
- 실제 서빙 설정/모델은 `doc_parser` 레포의 `genon/serving/paddle/`(이미 검증된 레퍼런스)에서 가져옴:
  - `config/ocr.yaml`: 한국어 인식(`korean_PP-OCRv5_mobile_rec`) + `PP-OCRv5_server_det`,
    `model_dir`을 `/models/paddleocr_model/...`로 고정.
  - `resources/paddleocr_model.zip`(87MB, 검출+한국어 인식 가중치)을 이미지 빌드 시 `/models`에 구워둠.
  - `pyproject.toml`/`uv.lock`: `paddlepaddle-gpu==3.2.0` + `paddlex==3.3.6` + `paddlex[serving,ocr]`,
    `extra-index-url`로 paddle 전용 인덱스(cu126) 사용 — GPU 전용 빌드 (로컬 CPU 테스트와는 별개).
  - `etc/smoke_test_compare.py`도 함께 이식 (baseline vs current 결과 비교용, 필요시 사용).
- 이식 중 레퍼런스에 있던 버그 수정: `supervisord.conf`의 program 이름(`paddlex-ocr`)과
  `health_checker.sh`의 restart 대상 이름(원래 `paddlex`)이 불일치해서 헬스체크 실패 시 자동 재시작이
  안 먹는 문제였음 → `health_checker.sh`가 `paddlex-ocr`을 재시작하도록 수정.
- k8s manifest 이름(`doc-parser-ocr-*`)은 레퍼런스 그대로 유지 (GenOS 플랫폼 내 다른 doc-parser-* 서비스와
  네이밍 일관성 위해).
- 아직 안 한 것: 실제 GPU 노드에서 이미지 빌드 + smoke_test.sh 구동 검증 (로컬은 CPU라 GPU 어설션이
  의도적으로 실패함).
