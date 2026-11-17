---
title: "[Local VLM] Mac M5 vs RTX 3090 Ti — 27B급 로컬 비전 모델 영상 추론 성능 비교"
description: "Mac M5와 RTX 3090 Ti에서 수행한 로컬 영상 추론의 처리 시간과 환경 차이를 비교합니다."
pubDate: "Oct 18 2026"
---

# [Local VLM] Mac M5 vs RTX 3090 Ti — 27B급 로컬 비전 모델 영상 추론 성능 비교

## 먼저 결론부터

동일한 모바일 화면 녹화 영상을 분석한 실행에서 Apple M5 MacBook + MLX-VLM 구성은 약 376.4초, Linux RTX 3090 Ti + llama.cpp CUDA 구성은 약 99.7초가 걸렸다. Linux 구성이 약 3.8배 빠른 결과다. 다만 모델 실행 엔진, 양자화, 프레임 입력 방식이 서로 달라 이 결과를 하드웨어만의 성능 차이로 볼 수는 없다.

이 결과를 “RTX 3090 Ti가 M5보다 언제나 3.8배 빠르다”로 일반화할 수는 없다. 양쪽은 같은 크기의 모델 계열이지만 양자화된 가중치 파일, 추론 엔진, 프레임 입력 방식이 다르다. 공개 초안에서는 특정 모델명을 생략한다.

## 하드웨어와 실행 환경

| 구분 | Apple Silicon 환경 | NVIDIA 환경 |
| --- | --- | --- |
| 장치 | Apple M5 MacBook | NVIDIA GeForce RTX 3090 Ti |
| 메모리 | 통합 메모리 32GB | GPU 메모리 24,564 MiB 기록, 제품 사양 24GB GDDR6X |
| 메모리 대역폭 | 약 153 GB/s | 약 1008 GB/s |
| 운영체제 | macOS | Ubuntu 22.04.4 |
| GPU 소프트웨어 | MLX / MLX-VLM | NVIDIA 드라이버 550.163.01, CUDA 드라이버 지원 12.4 |
| 컨테이너 | 미사용 기록 | Docker 27.3.1 |

Apple은 M5 통합 메모리 대역폭을 153GB/s로 발표했다. NVIDIA 제품 사양은 RTX 3090 Ti를 24GB GDDR6X로 표기하고, GPU 아키텍처 자료는 메모리 대역폭을 1008GB/s로 제시한다. 이는 장치 사양 참고값이며 이 실험의 실제 메모리 대역폭 측정치는 아니다. ([Apple M5 발표](https://www.apple.com/newsroom/2025/10/apple-unleashes-m5-the-next-big-leap-in-ai-performance-for-apple-silicon/), [NVIDIA RTX 3090 Ti 사양](https://www.nvidia.com/en-us/geforce/graphics-cards/30-series/rtx-3090-3090ti/), [NVIDIA GPU 아키텍처 자료](https://images.nvidia.com/aem-dam/Solutions/Data-Center/l4/nvidia-ada-gpu-architecture-whitepaper-V2.02.pdf))

## 모델과 입력 경로

| 구분 | Mac | Linux |
| --- | --- | --- |
| 모델 표기 | 27B급 로컬 비전 모델 | 27B급 로컬 비전 모델 |
| 양자화 | MLX 4-bit | UD-Q4_K_M |
| 런타임 | MLX-VLM 0.7.1, MLX 0.32.2 | llama.cpp CUDA |
| 영상 입력 | 저장소의 Mac runner는 ffprobe·FFmpeg로 RGB 프레임과 영상 메타데이터 구성 | 실험 기록 기준: OpenCV 샘플링, FFmpeg `libx264rgb` / `crf 0` / `rgb24`, CFR 2 FPS |
| 프레임 구성 | 2 FPS 기준 | 76개 샘플 프레임에 최초 프레임 중복을 더한 78개 입력 프레임 |

두 환경은 서로 다른 배포본과 양자화 형식을 사용하므로 동일한 가중치 파일이라고 볼 수 없다. MLX-VLM은 Apple Silicon용 MLX 기반 VLM 런타임이고, llama.cpp는 멀티모달 GGUF 입력을 지원한다. 런타임이 다른 만큼 Python 호출 방식과 영상 해석 전 입력 경로도 달라진다. ([MLX-VLM](https://github.com/Neywa-Labs/mlx-vlm), [llama.cpp 멀티모달 문서](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md))

이번 연동은 바이브 코딩 방식으로 Python 코드를 구성하며 진행했다. 하지만 코드를 빠르게 작성한 것과 벤치마크가 공정하다는 것은 별개의 문제다. 모델 실행 엔진, 양자화, 입력 경로가 다른 점을 함께 설명하고, 약 3.8배는 해당 실행 결과로만 제시한다.

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

Linux API 실행은 HTTP 200으로 완료됐고, 입력 토큰 13,350개와 생성 토큰 1,700개, 추출 결과 36개를 반환했다. Mac의 토큰 수는 비교 자료에 없어 초당 토큰 처리량은 비교하지 않았다.

## 왜 차이가 났을까

RTX 3090 Ti는 개별 GDDR6X 메모리와 높은 대역폭을 갖고, M5는 CPU·GPU가 통합 메모리를 공유한다. 그러나 대역폭만으로 관찰된 3.8배 차이의 원인을 정량적으로 설명할 수 없다. 모델 연산 커널, 비전 인코더 실행, 프레임을 전달하는 방식, 토큰 생성 속도, 런타임 구현이 모두 다를 수 있다.

초기 Linux 시도에서는 다른 27B 실행 구성이 INT4 가중치를 적재한 다음 첫 텍스트 추론에서 GPU 메모리 부족으로 중단됐다. 이후 llama.cpp와 GGUF 양자화 모델을 사용하는 별도 실행 구성으로 RTX 3090 Ti에서 영상 추론을 완료했다. 서로 다른 로딩·추론 구성이므로 같은 설정을 조금 조정해 성공했다고 설명하지 않는다.

| 비교 항목 | 관찰 결과 | 해석 범위 |
| --- | --- | --- |
| 처리 시간 | Mac 약 376.4초, Linux 약 99.7초 | 이번 실행에서 관찰한 소요 시간 |
| 계산 비율 | 376.4 ÷ 99.7 ≈ 3.78 | Linux 구성이 약 3.8배 빠른 결과 |
| 하드웨어만의 성능 차이 | 엔진·양자화·입력 경로도 다름 | 이번 비교만으로 분리할 수 없음 |

## 측정 한계

| 한계 | 영향 |
| --- | --- |
| 엔진 차이 | MLX-VLM과 llama.cpp CUDA의 실행 경로가 다름 |
| 양자화 차이 | MLX 4-bit와 GGUF UD-Q4_K_M의 표현 방식이 다름 |
| 입력 차이 | Mac의 프레임·메타데이터 전달과 Linux CFR 영상 구성이 다름 |
| 표본 수 제한 | 반복 실행 분포, 평균, 표준편차 미측정 |
| 정확도 기준 제한 | 36/36 출력 간 대응은 독립 정답지와의 비교가 아니므로 영상 기반 정답률을 의미하지 않음 |

하드웨어 차이를 분리해 비교하려면 입력 프레임, 프롬프트, 출력 제한, 측정 구간을 통일하고 여러 차례 반복해야 한다. 이번 글은 한 번의 실행에서 관찰한 처리 시간을 비교하며, 평균이나 일반적인 성능 우위를 뜻하지 않는다.

이번 실행에서는 RTX 3090 Ti 기반 Linux 구성이 M5 Mac 구성보다 약 3.8배 빨리 분석을 마쳤다. 이 결과에는 하드웨어뿐 아니라 엔진, 양자화, 입력 경로의 차이도 반영되어 있다.

다음 편에서는 두 실행 결과의 개수가 같아도 항목별 출력이 달랐던 관찰과, Python·NumPy로 모델 입력 전 프레임 보존을 확인한 방법을 다룬다.

## 참고 문서

- [Apple의 M5 발표 및 메모리 대역폭](https://www.apple.com/newsroom/2025/10/apple-unleashes-m5-the-next-big-leap-in-ai-performance-for-apple-silicon/)
- [NVIDIA RTX 3090 Ti 제품 사양](https://www.nvidia.com/en-us/geforce/graphics-cards/30-series/rtx-3090-3090ti/)
- [NVIDIA GPU 아키텍처 자료](https://images.nvidia.com/aem-dam/Solutions/Data-Center/l4/nvidia-ada-gpu-architecture-whitepaper-V2.02.pdf)
- [MLX-VLM](https://github.com/Neywa-Labs/mlx-vlm)
- [llama.cpp 멀티모달 문서](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md)
