# PaddleOCR Server

PaddleX `OCR` 파이프라인(텍스트 검출 + 방향 분류 + 언워핑 + 한국어 인식 모델 교체)을
`paddlex --serve`로 감싼 서빙 앱입니다. 이미지를 보내면 인식된 텍스트(text/score/bbox)를 반환합니다.

## Quick start

```shell
uv sync --index-strategy unsafe-best-match
uv run paddlex --serve --pipeline config/ocr.yaml --host 0.0.0.0 --port 8080
```

`config/ocr.yaml`의 `TextDetection`/`TextRecognition`은 `model_dir`이
`/models/paddleocr_model/...`로 고정되어 있습니다. 로컬에서 그대로 띄우려면
`resources/paddleocr_model.zip`을 `/models/`에 풀어두거나(`unzip resources/paddleocr_model.zip -d /models`),
`model_dir: null`로 바꿔 PaddleX가 첫 요청 시 자동 다운로드하게 두면 됩니다.

## Environment

| Name | Default | Description |
|---|---|---|
| `OCR_PORT` | `8080` | health checker가 확인할 포트. |

## API

PaddleX serve는 내부적으로 FastAPI 기반이며 다음 엔드포인트를 제공합니다.

### `GET /health`

```json
{"errorCode": 0, "errorMsg": "Success"}
```

### `POST /ocr`

```json
{"file": "<base64 이미지/PDF>", "fileType": 1}
```

### 호출 예시 (PyMuPDF로 페이지 렌더링 → 요청)

```python
import base64
import fitz
import httpx

doc = fitz.open("sample.pdf")
page = doc[0]
png_bytes = page.get_pixmap(dpi=150).tobytes("png")

resp = httpx.post(
    "http://localhost:8080/ocr",
    json={"file": base64.b64encode(png_bytes).decode("ascii"), "fileType": 1},
)
print(resp.json())
```

## Build & deploy

- `build-script/paddle-ocr-build.sh` (+ `paddle-ocr-build.config`): 도커 이미지 빌드/푸시.
- `k8s-manifest/`: `llmops` 네임스페이스용 Deployment/Service (기본은 ClusterIP, `-node-port` 버전은 NodePort).
- `docker/Dockerfile`: base → deps(uv sync) → models(모델 zip 압축 해제) → runtime 멀티스테이지.
  `supervisord`가 `paddlex --serve`를 실행하고, `etc/health_checker.sh`가 60초마다 `/health`를 확인해
  실패 시 자동 재시작합니다.
- `etc/smoke_test.sh`: 컨테이너 내부에서 `health` → `inference` 순으로 검증. GPU 미노출 환경에서는
  의도적으로 실패합니다. `compare` 서브커맨드로 이전 결과(JSON)와 텍스트/score 근사 비교도 가능합니다.

레퍼런스: `doc_parser` 레포의 `genon/serving/paddle/`, 그리고 레이아웃 서빙 레포([`detr`](../detr))의 구조.
