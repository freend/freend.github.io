---
title: "[Local VLM] Python으로 구축한 로컬 멀티모달 LLM 영상 분석 파이프라인"
description: "Python으로 영상 전처리, 로컬 멀티모달 추론, JSON 검증을 연결한 실험 기록입니다."
pubDate: "Oct 10 2026"
---

# [Local VLM] Python으로 구축한 로컬 멀티모달 LLM 영상 분석 파이프라인

## 먼저 결론부터

37.678초 길이의 모바일 화면 녹화 영상을 Python으로 처리해, 화면에 시간에 따라 표시되는 목록 항목을 로컬 멀티모달 LLM으로 분석하고 구조화된 JSON으로 추출했다. Mac에서는 MLX-VLM으로 샘플 프레임을 전달했고, Linux에서는 OpenCV와 FFmpeg로 입력 영상을 구성한 뒤 llama.cpp CUDA 추론 API에 연결했다.

이번 실험에서 중요한 부분은 모델 이름보다 영상 입력을 추론 엔진이 받을 수 있는 형태로 만들고, 응답을 다시 파싱 가능한 데이터로 검증하는 연결부였다. 실제 Python 구현을 확인해 보니 Mac 쪽은 `ffprobe`와 FFmpeg로 프레임을 읽고 `mlx_vlm.stream_generate`에 전달한다. Linux 쪽은 Python 어댑터가 영상 파일을 multipart HTTP 요청으로 보내고 응답 구조를 확인한다. 아래 코드는 이 구현에서 공개 가능한 부분을 발췌했으며, 프로젝트 전용 데이터 필드와 내부 경로는 제외했다. Linux 쪽 프레임 인코딩은 실행 기록에 남은 설정을 기준으로 설명한다.

두 환경에는 SHA-256이 일치한 동일 원본 영상이 사용됐다. 보존된 Linux 실행 기록에서는 원시 API 응답과 저장된 36개 항목의 순서까지 맞는 것이 확인됐다. 이 연결 덕분에 보존된 Mac 결과와 기존에 수행한 개별 항목 비교를 Linux 저장 결과에 대응시킬 근거가 생겼다. 항목별 검증 수치는 3편에서 다룬다.

## 왜 텍스트 입력만으로 처리하지 않았나

입력은 스크롤에 따라 화면 구성이 바뀌는 모바일 화면 녹화 영상이다. 텍스트를 미리 추출해 모델에 넣는 방식만으로는 항목이 화면에 나타나는 시점, 주변 UI와의 관계, 같은 항목의 반복 노출을 함께 다루기 어렵다. 그래서 영상의 프레임을 모델 입력으로 사용하고, 시각 정보와 읽을 수 있는 텍스트를 함께 구조화하도록 했다.

이번 작업은 객체 탐지나 실시간 스트리밍이 아니다. 일정 간격으로 뽑은 프레임에서 보이는 여러 항목을 한 번의 분석 입력으로 구성하고, 중복 노출이 있는 화면에서 항목 정보를 정리하는 실험이다.

| 입력 항목 | 값 |
| --- | --- |
| 영상 종류 | 모바일 화면 녹화 |
| 길이 | 약 37.678초 |
| 해상도 | 1126 × 2436 |
| 코덱 | HEVC |
| 샘플링 기준 | 약 2 FPS |
| 원본에서 샘플링한 프레임 | 76개 |
| 추출 결과 | 각 환경에서 36개 항목 |

## 전체 파이프라인

```text
HEVC 원본 영상
  -> Python에서 ffprobe로 영상 메타데이터 확인
  -> FFmpeg로 시간 간격에 맞춰 RGB 프레임 디코딩
  -> 환경별 입력 구성
       Mac: 프레임 이미지와 메타데이터를 MLX-VLM 입력으로 구성
       Linux: 프레임 시퀀스를 FFmpeg로 CFR 영상으로 인코딩
  -> 27B급 로컬 비전 모델 추론
  -> JSON 텍스트 응답
  -> Python JSON 파싱 및 구조 검증
```

| 환경 | 모델 구성 | 추론 엔진 | 영상 입력 처리 |
| --- | --- | --- | --- |
| Apple M5 MacBook, 통합 메모리 32GB | 27B급 비전 모델, MLX 4-bit | MLX-VLM 0.7.1 / MLX 0.32.2 | Python이 ffprobe·FFmpeg를 호출해 RGB 프레임과 영상 메타데이터를 구성 |
| Ubuntu 22.04.4, RTX 3090 Ti | 27B급 비전 모델, GGUF `UD-Q4_K_M` | llama.cpp CUDA | 실험 기록 기준: Python·OpenCV 프레임 샘플링, FFmpeg CFR 2 FPS 입력 구성, HTTP API 추론 |

Mac 실행에는 최대 생성 토큰 2,048개 설정이 사용됐다. 보존된 Linux 실행 기록에서 실제 생성된 토큰은 1,700개다. 설정 상한과 실제 생성량은 서로 다른 값이며, 두 환경의 토큰 수 비교 자료로 사용하지 않는다.

두 환경에서는 같은 크기의 멀티모달 모델 계열을 사용했지만 가중치 배포본, 양자화, 전처리, 런타임은 다르다. 따라서 이 비교는 같은 모델 파일을 다른 기기에서 실행한 실험이 아니다. 모델 체크포인트 이름은 공개 초안에서 생략했다. MLX-VLM은 [MLX-VLM 저장소](https://github.com/Neywa-Labs/mlx-vlm)의 Python 추론 도구이고, llama.cpp는 [멀티모달 입력과 서버 API](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md)를 제공한다.

## 바이브 코딩으로 구현하고, 실제 실행으로 확인했다

이번 파이프라인은 바이브 코딩 방식으로 구현했다. 영상 파일을 프레임으로 만들고, 로컬 추론 엔진을 호출한 뒤, 결과를 구조화 데이터로 확인한다는 목표를 먼저 정리하고 Python 연결 코드를 반복해서 만들고 다듬었다. 모델 설치에서 끝내지 않고 Mac의 MLX-VLM 호출, Linux 추론 API 연결, JSON 응답 처리까지 한 흐름으로 묶는 작업이었다.

바이브 코딩으로 작성한 코드나 모델이 반환한 JSON을 그대로 정답으로 간주하지는 않았다. 입력 영상의 SHA-256, 프레임 수와 순서, HTTP 상태, 생성 토큰과 파싱 완료 여부를 확인했다. 두 환경의 결과도 항목 단위로 대조했다. AI 도구는 구현 속도를 높였지만, 실험 결과의 근거는 실행 자료와 별도 검증에서 가져왔다.

## Mac에서 ffprobe와 FFmpeg로 프레임 샘플링

Mac 측 실제 runner는 OpenCV의 `VideoCapture` 대신 `ffprobe`로 길이·크기·프레임레이트를 읽고, 계산한 샘플 시각에 맞춰 FFmpeg로 RGB 프레임을 디코딩한다. 아래는 저장소 구현에서 FFmpeg 명령과 프레임 버퍼를 만드는 핵심 부분을 발췌한 코드다. `meta`에는 앞 단계에서 확인한 영상 정보가 들어 있고, 오류 검증과 타임스탬프 대조는 본문 길이를 위해 생략했다.

```python
rate = format(sampling_fps, ".12g")
command = [
    ffmpeg, "-nostdin", "-hide_banner", "-loglevel", "info", "-i", str(video),
    "-map", "0:v:0", "-an",
    "-vf", f"fps={rate}:start_time=0:round=up,showinfo",
    "-fps_mode", "passthrough", "-frames:v", str(frame_count),
    "-pix_fmt", "rgb24", "-f", "rawvideo", "pipe:1",
]
process = subprocess.Popen(
    command, stdout=subprocess.PIPE, stderr=subprocess.PIPE, bufsize=0,
)
assert process.stdout is not None and process.stderr is not None
stderr_result = []
stderr_thread = threading.Thread(
    target=lambda: stderr_result.append(process.stderr.read()), daemon=True,
)
stderr_thread.start()

bytes_per_frame = width * height * 3
frames = np.empty((frame_count, 3, height, width), dtype=np.uint8)
decoded = 0
for index in range(frame_count):
    buffer = bytearray()
    while len(buffer) < bytes_per_frame:
        part = process.stdout.read(
            min(1024 * 1024, bytes_per_frame - len(buffer))
        )
        if not part:
            break
        buffer.extend(part)
    if len(buffer) != bytes_per_frame:
        raise RuntimeError("샘플 프레임을 끝까지 디코딩하지 못했습니다.")
    frames[index] = (
        np.frombuffer(buffer, dtype=np.uint8)
        .reshape(height, width, 3)
        .transpose(2, 0, 1)
    )
    decoded += 1

extra = process.stdout.read(1) if decoded == frame_count else b""
process.stdout.close()
return_code = process.wait()
stderr_thread.join()
if return_code != 0 or decoded != frame_count or extra:
    raise RuntimeError("영상 프레임 디코딩이 끝나지 않았거나 추가 데이터가 있습니다.")
```

실제 구현은 FFmpeg stderr의 `showinfo` 타임스탬프를 요청한 샘플 시각과 대조하고, 디코딩한 프레임 수가 예상과 같은지도 확인한다. 프레임 배열은 RGB 채널 기준 `(프레임, 채널, 높이, 너비)` 순서로 만든다. Mac runner에는 첫 샘플 프레임 두 장을 앞에 추가해 MLX-VLM 영상 메타데이터와 함께 전달하는 단계도 있다. 이 처리가 해당 보존 실행에 적용됐는지는 실행별 원시 메타데이터와 구분해 확인해야 한다.

## Linux 입력: FFmpeg로 RGB 무손실 영상 구성

Linux 입력 구성 기록에는 `libx264rgb`, `crf 0`, `rgb24`, CFR 2 FPS가 남아 있다. 원본에서 샘플링한 76개 프레임에 최초 프레임을 두 번 추가해 78개 프레임으로 만들었고, PTS가 약 1초 이동했다. 이 값은 Linux 추론 서버의 실험 설정으로 확인한 사실이다. 해당 서버의 인코딩 소스는 이 저장소에 없어, 위 설정을 실제 Python 함수 코드인 것처럼 재구성하지 않았다. 프레임 수와 순서가 유지되어도 llama.cpp 내부의 이미지 변환이나 모델 해석까지 동일하다는 뜻은 아니다.

## 추론 API와 JSON 파싱

Linux API 호출 코드는 저장소에서 확인했다. Python 표준 라이브러리의 `urllib.request`로 multipart 요청을 보내고, 제한 시간을 둔 뒤 JSON 응답을 파싱한다. 아래는 실제 어댑터의 요청 처리에서 내부 주소와 경로를 제외한 발췌다. multipart 본문은 앞 단계에서 구성하며, 응답은 뒤에서 성공 여부와 결과 구조를 확인한다.

```python
import json
from urllib.request import Request, urlopen


request = Request(
    f"{base_url}{endpoint}",
    data=multipart_body,
    headers={
        "Accept": "application/json",
        "Content-Type": content_type,
    },
    method="POST",
)
with urlopen(request, timeout=timeout_seconds) as response:
    response_body = response.read()

payload = json.loads(response_body)
```

실제 어댑터는 HTTP 오류·시간 초과·연결 실패를 각각 분류하고, JSON 파싱 성공 뒤 필수 결과 구조를 검사한다. 비공개 목적을 드러낼 수 있는 결과 필드명과 예외 타입은 코드에서 제외했다. 보존된 Linux 실행 기록에는 입력 토큰 13,350개, 생성 토큰 1,700개, `finish_reason=stop`, `parse_complete=true`, `incomplete_tail=false`가 남아 있다. 이 수치는 Linux 실행 자료이며 Mac 실행 토큰 수와 직접 비교할 자료는 없다.

Mac 쪽 runner는 MLX-VLM의 `stream_generate` 결과를 조각 단위로 모은 뒤 파서에 넘기고, 원문과 구조화 결과를 별도 파일로 기록한다. 아래는 실제 생성 호출과 텍스트 수집 부분이다. `media_kwargs`에는 앞서 만든 프레임 배열 및 영상 메타데이터가 전달된다.

```python
chunks: list[str] = []
final_response = None
for response in stream_generate(
    model,
    processor,
    generation_prompt,
    **media_kwargs,
    max_tokens=max_tokens,
):
    text = getattr(response, "text", None)
    if not isinstance(text, str):
        raise RuntimeError("모델 응답에 텍스트 조각이 없습니다.")
    chunks.append(text)
    final_response = response

raw_text = "".join(chunks)
```

애플리케이션 어댑터는 파서가 만든 JSON 파일을 읽은 뒤 최상위 값이 객체 배열인지 확인한다. 도메인 필드가 드러나는 파서 호출과 파일 경로는 공개 코드에서 생략했다.

```python
records = json.loads(result_file.read_text(encoding="utf-8"))
if not isinstance(records, list) or any(
    not isinstance(record, dict) for record in records
):
    raise ValueError("추론 결과의 구조가 올바르지 않습니다.")
```

구조 예시는 다음과 같다. **실제 추론 결과가 아니라 구조를 설명하기 위한 가상 예시**이며, 실제 데이터와 프로젝트 필드명을 사용하지 않았다.

```json
[
  {
    "label": "항목 A",
    "visible_text": "화면에서 읽힌 짧은 문구",
    "visual_note": "아이콘과 텍스트가 함께 표시됨"
  }
]
```

## 구현에서 확인한 것과 남은 것

| 확인된 기록 | 이 정보만으로는 확인할 수 없는 것 |
| --- | --- |
| Mac runner의 ffprobe·FFmpeg 샘플링 및 MLX-VLM 호출 코드 확인 | Linux 측 프레임 인코딩은 기록된 설정 범위에서 설명 |
| Linux Python 어댑터의 multipart HTTP 전송과 응답 구조 검사 코드 확인 | API 내부 주소·경로와 실제 데이터 필드는 공개하지 않음 |
| Linux 원시 API 응답과 저장된 36개 항목의 순서 일치 | 36개 항목이 실제 영상의 독립 정답지와 모두 일치하는지 |
| Linux 응답 파싱 완료 상태 기록 | 원시 실행 시간과 Mac 토큰 수 |

실험을 통해 Python이 영상 파일을 프레임과 모델 입력으로 바꾸고, 별도 추론 런타임을 HTTP로 호출하며, 결과를 구조화된 형태로 다시 검증하는 접착 계층 역할을 한다는 점을 확인했다. 바이브 코딩으로 이 연결을 빠르게 구성하더라도 입력 형식, 타임스탬프, 응답 파싱을 직접 확인해야 파이프라인의 동작을 설명할 수 있었다.

다음 편에서는 같은 영상을 처리한 Mac M5와 RTX 3090 Ti 구성의 실행 시간과 추론 환경 차이를 비교한다.

## 참고 문서

- [MLX-VLM 공식 저장소](https://github.com/Neywa-Labs/mlx-vlm)
- [llama.cpp 멀티모달 문서](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md)
- [OpenCV Python 비디오 가이드](https://docs.opencv.org/4.x/dd/d43/tutorial_py_video_display.html)
- [FFmpeg 공식 문서](https://ffmpeg.org/ffmpeg.html)
