---
title: "[Local VLM] Mac M5 vs RTX 3090 Ti — 27B급 로컬 비전 모델 영상 추론 성능 비교"
description: "Mac M5와 RTX 3090 Ti에서 수행한 로컬 영상 추론의 처리 시간과 환경 차이를 비교합니다."
pubDate: "Oct 18 2026"
---

## 먼저 결론부터

동일한 모바일 화면 녹화 영상을 분석했을 때 Apple M5 MacBook + MLX-VLM 구성은 376.4초, Linux RTX 3090 Ti + llama.cpp CUDA 구성은 99.7초가 걸렸다. 이번 실행에서는 Linux 구성이 약 3.8배 빨랐고, 양쪽에서 각각 36개 항목을 추출했다. 두 환경은 모델 실행 엔진, 양자화, 프레임 입력 방식이 달라 이 결과는 해당 구성의 실행 시간 비교로 해석해야 한다.

같은 개수의 항목이 추출됐다는 사실만으로 항목별 내용까지 같다고 볼 수는 없다. 결과의 일관성과 프레임 보존은 [후속 글](/blog/local-vlm-output-consistency-and-frame-validation/)에서 따로 확인한다.

## 하드웨어와 실행 환경

| 구분 | Apple Silicon 환경 | NVIDIA 환경 |
| --- | --- | --- |
| 장치 | Apple M5 MacBook | NVIDIA GeForce RTX 3090 Ti |
| 메모리 | 통합 메모리 32GB | GPU 메모리 24,564 MiB 기록, 제품 사양 24GB GDDR6X |
| 메모리 대역폭 | 약 153 GB/s | 약 1008 GB/s |
| 운영체제 | macOS | Ubuntu 22.04.4 |
| GPU 소프트웨어 | MLX / MLX-VLM | NVIDIA 드라이버 550.163.01, CUDA 드라이버 지원 12.4 |
| 컨테이너 | 미사용 기록 | Docker 27.3.1 |

Apple과 NVIDIA의 메모리 대역폭은 각 제조사 자료에 공개된 장치 사양이다. 이번 실행에서 실제 대역폭을 측정한 값은 아니다. ([Apple M5 발표](https://www.apple.com/newsroom/2025/10/apple-unleashes-m5-the-next-big-leap-in-ai-performance-for-apple-silicon/), [NVIDIA RTX 3090 Ti 사양](https://www.nvidia.com/en-us/geforce/graphics-cards/30-series/rtx-3090-3090ti/), [NVIDIA GPU 아키텍처 자료](https://images.nvidia.com/aem-dam/Solutions/Data-Center/l4/nvidia-ada-gpu-architecture-whitepaper-V2.02.pdf))

## 모델과 입력 경로

| 구분 | Mac | Linux |
| --- | --- | --- |
| 모델 표기 | 27B급 로컬 비전 모델 | 27B급 로컬 비전 모델 |
| 양자화 | MLX 4-bit | UD-Q4_K_M |
| 런타임 | MLX-VLM 0.7.1, MLX 0.32.2 | llama.cpp CUDA |
| 영상 입력 | 저장소의 Mac runner는 ffprobe·FFmpeg로 RGB 프레임과 영상 메타데이터 구성 | 실험 기록 기준: OpenCV 샘플링, FFmpeg `libx264rgb` / `crf 0` / `rgb24`, CFR 2 FPS |
| 프레임 구성 | 2 FPS 기준 | 76개 샘플 프레임에 최초 프레임 중복을 더한 78개 입력 프레임 |

두 환경은 같은 27B급 모델 계열을 사용했지만 양자화된 가중치와 추론 엔진은 달랐다. Mac은 Apple Silicon용 MLX-VLM 런타임과 MLX 4-bit 모델을, Linux는 llama.cpp CUDA와 GGUF UD-Q4_K_M 모델을 사용했다. 영상 프레임 구성도 표와 같이 달라 동일한 가중치 파일이나 입력 경로를 비교한 실험은 아니다. ([MLX-VLM](https://github.com/Neywa-Labs/mlx-vlm), [llama.cpp 멀티모달 문서](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md))

Python 연동은 AI 코딩 에이전트를 활용한 바이브 코딩 방식으로 구성했다. 실행 환경의 차이를 확인하고 기록해, 결과를 이번 구성에서 관찰한 값으로 해석할 수 있도록 정리했다.

## 처리 시간

| 항목 | Mac M5 | RTX 3090 Ti |
| --- | ---: | ---: |
| 영상 길이 | 약 37.7초 | 약 37.7초 |
| 처리 시간 | 약 376.4초 | 약 99.7초 |
| 추출 결과 수 | 36개 | 36개 |
| 입력 토큰 | 비교 자료 없음 | 13,350개 |
| 생성 토큰 | 비교 자료 없음 | 1,700개 |
| 완료 상태 | 완료 | `stop`, JSON 파싱 완료 |

```text
실행 시간 비교 (각 █ ≈ 12.5초)

Mac M5       ██████████████████████████████  376.4초
RTX 3090 Ti  ████████                         99.7초
```

두 실행 시간의 산술 비율은 다음과 같다.

```text
376.4초 ÷ 99.7초 = 3.775... ≈ 3.8배
```

Linux API 실행은 HTTP 200으로 완료됐다. Mac의 토큰 수 비교 자료가 없어 초당 토큰 처리량은 비교하지 않았다.

## 왜 차이가 났을까

RTX 3090 Ti는 개별 GDDR6X 메모리를 사용하고, M5는 CPU와 GPU가 통합 메모리를 공유한다. 메모리 구조는 두 환경의 차이 중 하나지만, 이번 기록만으로 처리 시간 차이를 대역폭 하나에 정량적으로 연결할 수는 없다. 모델 연산 커널, 비전 인코더, 프레임 전달, 토큰 생성과 런타임 구현도 실행 시간에 영향을 줄 수 있다.

초기 Linux 실험에서는 별도의 27B 실행 구성이 INT4 가중치를 적재한 뒤 첫 텍스트 추론에서 GPU 메모리 부족으로 멈췄다. 이후 llama.cpp와 GGUF 양자화 모델을 사용하는 실행 구성으로 전환해 RTX 3090 Ti에서 영상 추론을 완료했다.

## 측정 범위

각 환경에서 한 번씩 실행했으며 반복 실행의 평균과 분산은 측정하지 않았다. 엔진, 양자화, 프레임 입력 방식도 달라 이번 결과는 두 실행 구성의 비교다.

두 환경에서 각각 36개를 추출했고, 결과 간 항목은 36/36으로 대응했다. 이 대응은 독립적인 정답 데이터와 대조한 정확도가 아니므로 영상의 추출 정확도를 뜻하지 않는다. 세부 비교 결과와 Python·NumPy로 모델 입력 전 프레임 보존을 확인한 과정은 다음 글에서 다룬다.

## 참고 문서

- [Apple의 M5 발표 및 메모리 대역폭](https://www.apple.com/newsroom/2025/10/apple-unleashes-m5-the-next-big-leap-in-ai-performance-for-apple-silicon/)
- [NVIDIA RTX 3090 Ti 제품 사양](https://www.nvidia.com/en-us/geforce/graphics-cards/30-series/rtx-3090-3090ti/)
- [NVIDIA GPU 아키텍처 자료](https://images.nvidia.com/aem-dam/Solutions/Data-Center/l4/nvidia-ada-gpu-architecture-whitepaper-V2.02.pdf)
- [MLX-VLM](https://github.com/Neywa-Labs/mlx-vlm)
- [llama.cpp 멀티모달 문서](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md)
