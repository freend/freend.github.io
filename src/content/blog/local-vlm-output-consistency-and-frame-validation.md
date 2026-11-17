---
title: "[Local VLM] 같은 영상, 다른 결과 — 멀티모달 LLM 출력 일관성 검증"
description: "동일 영상의 항목별 결과를 대조하고 Python으로 프레임 보존을 검증한 과정을 정리합니다."
pubDate: "Oct 31 2026"
---

# [Local VLM] 같은 영상, 다른 결과 — 멀티모달 LLM 출력 일관성 검증

## 추출 건수가 같으면 결과도 같은가

비교 기록에서는 같은 원본 영상의 SHA-256이 Mac과 Linux에서 일치하는 것을 확인했다. Mac 결과는 보존된 실행 자료에서 가져왔고, Linux 결과는 보존된 실제 API 응답에서 확보했다. Linux 원시 응답에 포함된 36개 항목과 저장된 36개 항목의 순서도 일치했다. 이 연결로 과거에 수행한 항목별 비교 결과를 보존된 Linux 결과에 대응시킬 근거를 확보했다.

| 검증 항목 | 결과 |
| --- | ---: |
| Mac 결과 항목 수 | 36개 |
| Linux 결과 항목 수 | 36개 |
| 개별 항목 일대일 대응 | 36/36 |
| 누락·추가 항목 | 0개 |
| 중복 초과 | 0개 |
| 모호한 매칭 | 0개 |
| 비교한 구조화 값 일치 | 36/36 |
| 텍스트 표기 차이 | 7개: 의미 있는 문자열 6개, 말줄임 표기 1개 |
| 기타 속성 표현 차이 | 1개 |

이 36/36은 **두 추론 결과 간의 일대일 대응**을 말한다. 실제 영상의 모든 항목을 별도의 정답지와 대조한 것은 아니므로 Recall 100%나 절대 정확도 100%로 해석할 수 없다. 문자열 차이 7개 중 어느 쪽이 정답인지도 이번 비교에서는 판정하지 않았다. 원시 결과와 내부 식별값은 공개하지 않는다.

## 화면 판독과 의미 정리는 하나의 단계가 아니다

36개 항목은 모두 대응했고 비교 대상으로 삼은 구조화 값도 모두 일치했다. 차이는 문자열 표기와 한 가지 기타 속성의 표현에서 확인됐다. 관찰된 한 사례에서는 Mac이 의미를 정리한 표현을 출력했고, Linux는 입력에 가까운 표현을 내놓았지만 일부 문자가 달랐다. 원문과 구체적인 속성명은 싣지 않는다.

| 가설 | 관찰 근거 | 결론 |
| --- | --- | --- |
| 샘플링·인코딩 과정에서 프레임이 달라졌다 | 대응 프레임 비교에서 채널 차이 0이 기록됨 | 비교한 전처리 프레임의 픽셀 손실은 확인되지 않음 |
| 문맥이 문자열·속성 표현에 영향을 주었다 | 전체/부분 화면 실험과 Mac/Linux 출력 차이 | 가능한 설명이지만 원인으로 확정하지 않음 |
| 모델이 화면 문구를 그대로 복사한다 | 의미 정규화 표현과 원문에 가까운 표현이 관찰됨 | 시각 판독과 의미 해석이 함께 작용할 수 있다는 가설 |
| 두 결과가 영상의 모든 항목을 빠짐없이 찾았다 | Mac/Linux 항목끼리 36/36 일대일 대응 | 독립 정답지가 없으므로 Recall은 검증되지 않음 |
| 문자열 중 한쪽이 정답이다 | 7개 차이 확인 | 어느 출력을 정답으로 볼지 판정하지 않음 |

관찰된 사실은 두 환경의 출력 문자열이 달랐다는 것이다. 의미 해석이나 주변 문맥이 영향을 주었을 수 있다는 것은 가설이며, 모델 내부의 실제 판단 과정은 확인하지 않았다. 차이를 단순한 OCR 오류 하나로 설명할 근거도 없다.

## Python으로 항목을 안전하게 대조하기

검증은 각 결과를 JSON으로 읽고, 두 결과에서 유일하게 대응하는 항목만 짝지은 뒤 비교하는 방식으로 설명할 수 있다. 값이 비슷하다는 이유만으로 서로 다른 항목을 강제로 합치면 안 된다. 아래 코드는 비교 절차를 보여주기 위해 재구성한 **설명용 예제**다. `comparison_id`, `value`, `text`, `attribute`는 의미를 드러내지 않는 예제용 임시 이름이며 실제 데이터 구조를 반영하지 않는다.

`comparison_id`는 양쪽 결과에서 같은 항목을 가리키고 각 결과 안에서 유일하다고 확인된 비교용 키를 뜻한다. 실제 데이터에 유일하고 안전한 키가 없거나 키가 중복되면 자동 매칭을 보류한다. 어떤 단일 값만으로 항목을 억지로 대응시키지 않고, 모호한 후보는 별도 집계한다.

```python
from collections import defaultdict
import json
from pathlib import Path


def load_rows(path: str) -> list[dict]:
    with Path(path).open(encoding="utf-8") as f:
        rows = json.load(f)
    if not isinstance(rows, list) or not all(isinstance(row, dict) for row in rows):
        raise ValueError("최상위 값은 JSON 객체 배열이어야 합니다.")
    return rows


def index_by_key(rows: list[dict]) -> dict[str, list[dict]]:
    index = defaultdict(list)
    for row in rows:
        key = row.get("comparison_id")
        if not isinstance(key, str) or not key:
            raise ValueError("각 항목에 비어 있지 않은 comparison_id가 필요합니다.")
        index[key].append(row)
    return index


def without_ellipsis(text: str) -> str:
    return text.rstrip().removesuffix("...").removesuffix("…").rstrip()


mac = index_by_key(load_rows("mac-result.json"))
linux = index_by_key(load_rows("linux-result.json"))
matched = []
missing_in_linux = 0
extra_in_linux = 0
ambiguous_keys = 0
duplicate_excess = 0

for key in sorted(mac.keys() | linux.keys()):
    left = mac.get(key, [])
    right = linux.get(key, [])
    duplicate_excess += max(0, len(left) - 1) + max(0, len(right) - 1)

    if len(left) > 1 or len(right) > 1:
        ambiguous_keys += 1
    elif not left:
        extra_in_linux += 1
    elif not right:
        missing_in_linux += 1
    else:
        matched.append((left[0], right[0]))

value_matches = 0
text_differences = 0
ellipsis_only_differences = 0
meaningful_text_differences = 0
attribute_differences = 0

for left, right in matched:
    value_matches += left["value"] == right["value"]
    attribute_differences += left["attribute"] != right["attribute"]

    left_text, right_text = left["text"], right["text"]
    if left_text != right_text:
        text_differences += 1
        if without_ellipsis(left_text) == without_ellipsis(right_text):
            ellipsis_only_differences += 1
        else:
            meaningful_text_differences += 1

summary = {
    "one_to_one": len(matched),
    "missing_in_linux": missing_in_linux,
    "extra_in_linux": extra_in_linux,
    "duplicate_excess": duplicate_excess,
    "ambiguous_keys": ambiguous_keys,
    "value_matches": value_matches,
    "text_differences": text_differences,
    "ellipsis_only_differences": ellipsis_only_differences,
    "meaningful_text_differences": meaningful_text_differences,
    "attribute_differences": attribute_differences,
}
print(summary)
```

위 코드는 비교 절차를 설명하기 위한 예제이며, 이번 검증에 사용된 실제 비교 스크립트의 발췌가 아니다. 검증 기록의 요약값은 표에 따로 적었다. 예제는 1:1 대응 여부와 값·문자열·기타 속성의 차이를 분리하고, 누락·추가·중복·모호한 후보를 드러내는 흐름을 보여준다. 비교 키가 적절한지 확인하지 않으면 집계 숫자만으로 매칭의 타당성을 주장할 수 없다.

## 전체 화면과 부분 화면 입력

Mac 로컬 비전 모델을 대상으로 전체 화면과 특정 항목만 남긴 부분 화면을 비교한 보조 실험에서는 전체 화면 입력이 8개 항목을 약 98.377초에, 부분 화면 입력이 1개 항목을 약 34.681초에 반환했다. 두 입력에서 같은 항목의 속성 표현도 달라졌다.

| 입력 구성 | 출력 수 | 보고된 시간 | 관찰 |
| --- | ---: | ---: | --- |
| 전체 화면 | 8개 | 약 98.377초 | 주변 화면 정보가 함께 포함됨 |
| 특정 항목만 남긴 화면 | 1개 | 약 34.681초 | 같은 항목의 속성 표현이 달라짐 |

입력 정보량과 추출 대상 수가 다르기 때문에 시간을 직접 비교하거나, 문맥만을 원인으로 확정할 수는 없다. 이 사례는 출력이 화면 구성에 민감할 수 있다는 추가 관찰로만 사용한다.

## 전처리 프레임은 보존됐는가

Linux 전처리 검증에서는 원본 영상에서 샘플링한 76개 프레임과 최초 프레임 중복을 포함한 78개 CFR 영상을 비교했다. 대응되는 76개 프레임에서 평균 RGB 차이는 0, 최대 채널 차이도 0으로 기록됐다. 순서가 유지됐고, 최초 프레임 중복으로 타임스탬프가 약 1초 이동한 사실도 확인됐다.

| 검증 항목 | 기록된 값 |
| --- | ---: |
| 샘플링된 원본 프레임 | 76개 |
| 추론용 CFR 영상 | 78개 |
| 대응 비교 프레임 | 76개 |
| 평균 채널 차이 | 0 |
| 최대 채널 차이 | 0 |
| 프레임 순서 | 대응 프레임에서 유지 |
| PTS 영향 | 최초 프레임 중복에 따른 약 1초 이동 |

아래는 같은 방식의 비교를 구현하는 **설명용 Python/NumPy 예제**다. 두 영상을 OpenCV로 디코딩한 뒤, 앞에 중복 프레임 두 장이 배치됐다는 가정 아래 뒤의 76개를 원본 샘플과 대응시킨다. 이 예제는 설명을 위해 작성했으며, 실제 픽셀 검증 코드와 프레임 대응 규칙은 확인하지 못했다.

```python
import cv2
import numpy as np


source = cv2.VideoCapture("sampled_source.mp4")
encoded = cv2.VideoCapture("llama_input.mp4")
if not source.isOpened() or not encoded.isOpened():
    raise OSError("비교할 영상 중 하나를 열 수 없습니다.")

try:
    # 인코딩 영상 앞부분에 중복된 첫 프레임 두 장을 건너뜁니다.
    for _ in range(2):
        ok, _ = encoded.read()
        if not ok:
            raise ValueError("중복 프레임 두 장을 읽지 못했습니다.")

    total_difference = 0
    compared_channels = 0
    maximum_difference = 0
    compared_frames = 0

    for index in range(76):
        ok_source, left = source.read()
        ok_encoded, right = encoded.read()
        if not ok_source or not ok_encoded:
            raise ValueError(f"프레임 {index}에서 입력 영상이 끝났습니다.")
        if left.shape != right.shape:
            raise ValueError(f"프레임 {index}의 크기가 다릅니다.")

        difference = np.abs(left.astype(np.int16) - right.astype(np.int16))
        total_difference += int(difference.sum(dtype=np.uint64))
        compared_channels += difference.size
        maximum_difference = max(maximum_difference, int(difference.max()))
        compared_frames += 1

    if source.read()[0] or encoded.read()[0]:
        raise ValueError("예상한 프레임 수보다 많은 프레임이 있습니다.")
finally:
    source.release()
    encoded.release()

print({
    "paired_frames": compared_frames,
    "mean_channel_difference": total_difference / compared_channels,
    "max_channel_difference": maximum_difference,
})
```

프레임 전체를 메모리에 보관하지 않고 한 쌍씩 비교하므로 큰 영상에서 불필요한 배열 복사를 줄인다. OpenCV는 BGR 배열을 반환하지만 양쪽을 같은 방식으로 디코딩해 채널별 차이를 비교한다. 평균과 최대 채널 차이가 모두 0이면 비교한 디코딩 프레임 배열은 픽셀 단위로 같다. 이는 FFmpeg 전처리 단계의 프레임 보존을 확인하는 증거다. llama.cpp 내부의 리사이즈·크롭·비전 인코더 입력이 동일했는지, 모델이 두 프레임을 같은 의미로 해석했는지는 확인하지 않는다.

## 사실과 미확인 영역

| 질문 | 확인 결과 | 추가로 필요한 검증 |
| --- | --- | --- |
| 전처리에서 픽셀이 바뀌었나? | 대응 프레임에서 차이 0이 보고됨 | 원시 프레임과 실제 비교 스크립트, 매핑 인덱스 확인 |
| 두 결과가 항목별로 대응하는가? | 일대일 대응 36/36, 누락·추가·중복·모호성 0 | 비교 키가 두 결과에서 같은 항목을 가리키는지 확인하는 절차가 필요 |
| 비교한 필드가 일치하는가? | 구조화 값 36/36, 텍스트 7·기타 속성 1 차이 | 독립 정답지에 대조한 결과는 아님 |
| 어느 결과가 영상에 맞는가? | 정답 라벨을 제공받지 못함 | 원본을 본 사람의 기준 라벨과 판정 절차 필요 |
| 주변 문맥이 영향을 줬나? | 전체/부분 화면 결과가 다름 | 같은 대상·동일 해상도·동일 프롬프트의 통제 실험 필요 |
| 모델 출력은 반복해도 같은가? | 반복 실행 자료 없음 | seed·샘플링 설정을 기록한 반복 실행 필요 |

정확도와 일관성을 더 확인하려면 프레임 파일 해시, 프롬프트, 모델 체크포인트와 해시, 생성 설정, 결과 JSON을 같은 실험 단위로 보관해야 한다. 그다음 민감한 내용을 노출하지 않는 익명 항목 ID로 결과를 맞추고, 화면 판독과 의미 정리를 별도 기준으로 채점해야 한다.

이번 비교 기록에서는 입력 원본을 SHA-256으로 연결하고, 두 결과의 36개 항목을 모두 일대일로 대조했다. 비교한 구조화 값은 모두 일치했고, 문자열 7개 및 기타 속성 1개에서 차이를 확인했다. Python·NumPy 프레임 검증은 전처리에서 대응 픽셀이 보존됐다는 근거를 보탰다. 이 결과도 독립 정답지와의 비교는 아니므로 영상 전체의 Recall이나 절대 정확도를 증명하지 않는다. 입력 보존 검증, 두 결과 간 일치 검증, 정답성 검증은 서로 다른 범위다.

이번 작업은 바이브 코딩으로 구현을 진행했지만, 빠르게 생성된 코드가 정확성을 대신해 주지는 않았다. 입력 해시, 결과 간 일대일 대조, 프레임 픽셀 비교처럼 각각 다른 질문에 답하는 검증 근거를 따로 봐야 했다. 설명용 코드와 실험 기록을 구분해 적어, 모델 결과를 과장하지 않고 검증된 범위를 전달할 수 있도록 했다.

## 참고 문서

- [OpenCV Python 비디오 가이드](https://docs.opencv.org/4.x/dd/d43/tutorial_py_video_display.html)
- [FFmpeg 공식 문서](https://ffmpeg.org/ffmpeg.html)
- [NumPy 배열 연산 문서](https://numpy.org/doc/stable/user/basics.ufuncs.html)
- [MLX-VLM 공식 저장소](https://github.com/Neywa-Labs/mlx-vlm)
- [llama.cpp 멀티모달 문서](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md)
