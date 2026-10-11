# D4D — RGB·열화상 기반 지뢰 후보 탐지 및 의미 정보 전송

<img width="2111" height="1194" alt="실내_추론시각화" src="https://github.com/user-attachments/assets/ebde49b1-aec8-47de-aaf4-1332a3b58bb5" />


## 프로젝트 개요

D4D는 RGB 카메라와 LWIR 열화상 센서를 함께 사용해 지뢰로 의심되는 후보 영역을 탐지하고, 후보의 시각적 특징을 구조화된 의미 정보로 변환해 원격 PC로 전달하는 프로젝트입니다.

단순히 이미지 전체를 전송하는 대신 엣지 장치에서 탐지와 해석을 수행한 뒤 필요한 결과만 전송하는 것을 목표로 합니다. 이를 통해 제한된 통신 환경에서도 탐지 위치, 노출 상태, 주변 지형과 같은 핵심 정보를 빠르게 확인할 수 있도록 구성했습니다.

> 이 프로젝트의 출력은 위험 물체에 대한 **탐지 보조 정보**입니다. 영상만으로 실제 지뢰 여부, 폭발 가능성 또는 작동 상태를 확정하지 않습니다.

## 전체 처리 흐름

```text
RGB 카메라 + LWIR 센서                         IMU 센서
          │                                      │
          ▼                                      ▼
센서 보정 및 RGB/LWIR 좌표 정합          위치·자세 데이터 수집
          │                              (X/Y, roll/pitch/yaw)
          ▼                                      │
4채널 RGB+LWIR 입력 생성                         │
          │                                      │
          ▼                                      │
YOLO 기반 지뢰 후보 탐지                         │
          │                                      │
          ├─ 바운딩 박스·신뢰도·추적 결과        │
          │                                      │
          ▼                                      │
후보 RGB crop → Qwen3-VL-2B                      │
          │                                      │
          ▼                                      │
구조화된 의미 정보(JSON)                         │
          │                                      │
          └──────────────┬───────────────────────┘
                         ▼
                Wi-Fi TCP 의미 정보 전송
                         │
                         ▼
               PC 수신 서버 및 웹 대시보드
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
     지뢰 후보·VLM 결과 표시     이동 경로·자세 표시
```

## 주요 기능

### 1. RGB·LWIR 센서 융합

- RGB 영상과 160×120 LWIR 영상을 보정값에 따라 동일한 좌표계로 정합
- 정합된 RGB 3채널과 LWIR 1채널을 결합해 4채널 YOLO 입력 생성
- 저장 이미지뿐 아니라 ROS 2의 `/cam0_stream1`, `/image_raw` 토픽을 이용한 실시간 입력 지원

### 2. YOLO 기반 지뢰 후보 탐지

- ONNX Runtime을 이용한 4채널 YOLO 추론
- 후보의 바운딩 박스, 클래스, 신뢰도 출력
- NMS 및 중복 후보 제거 적용
- 여러 프레임에서 동일 후보를 확인하는 추적 로직 구현
- 기본 추적 조건은 5프레임 중 3프레임 이상 확인된 후보이며, 확정된 후보에 대해서만 VLM을 호출하도록 구성

### 3. VLM 기반 후보 특징 분석

- 탐지된 후보의 RGB crop을 Qwen3-VL-2B 기반 모델에 입력
- 전체 장면을 다시 전송하지 않고 후보 중심 이미지로 의미 분석 수행
- 다음과 같은 관찰 정보를 구조화된 JSON으로 생성

  - 노출 상태(`exposure`)
  - 가림 요소(`occluder`)
  - 보이는 형태(`visible_shape`)
  - 보이는 색상(`visible_color`)
  - 주변 지형(`terrain`)
  - 관찰 한계(`observation_limits`)

- 허용된 enum과 필수 필드를 검사하는 schema 검증 및 제한적인 규칙 기반 보정 적용
- 모델이 후보를 실제 지뢰로 단정하거나 보이지 않는 재질·상태를 추측하지 않도록 프롬프트 구성

### 4. PC 및 엣지 장치 실행 경로

- Python 파이프라인으로 전처리·탐지·VLM·JSON 출력 검증
- C++17 파이프라인으로 이미지 처리, ONNX Runtime 추론, NMS, tracking, GenieX 호출 및 JSON 출력 통합
- PC에서는 ONNX Runtime CPU 또는 QNN 실행 경로 지원
- VENTUNO Q에서는 QNN Execution Provider를 이용한 YOLO 실행과 GenieX 기반 VLM 실행 경로 구성
- `GGML_HEXAGON_DEVICES=2`, `--compute npu` 설정을 이용한 Hexagon NPU 실행 및 로그 확인 절차 마련
- YOLO11n과 Qwen3-VL을 단일 C++ 프로세스에서 직접 연결하는 구현 진행

### 5. Wi-Fi 의미 정보 전송 및 대시보드

- TCP 기반 UTF-8 메시지 전송
- YOLO 탐지 수, 지뢰 후보 여부, VLM 의미 필드, schema 경고 및 추론 지연시간 전달
- 브라우저 대시보드에서 연결 상태, 최근 메시지, 경고 키워드 및 전송 속도 확인
- IMU 데이터를 함께 수신해 X/Y 이동 경로와 roll·pitch·yaw 표시
- 영상 원본 대신 의미 결과를 전송하는 저대역폭 통신 실험 환경 구성

## 지금까지의 진행 내용

| 구분 | 진행 내용 | 상태 |
|---|---|---|
| 데이터 | RGB/LWIR 지뢰 데이터 정리 및 대표 테스트 샘플 구성 | 진행 |
| 센서 보정 | RGB와 LWIR 좌표 정합 및 QAIPI 보정값 적용 | 구현 |
| 탐지 모델 | RGB+LWIR 4채널 YOLO ONNX 추론 | 구현 |
| 의미 분석 | Qwen3-VL-2B 후보 crop 분석 및 JSON schema 정의 | 구현 |
| 파이프라인 | Python 파이프라인을 C++17 실행 파일로 이식 | 구현 |
| 실시간 입력 | ROS 2 RGB/LWIR 토픽 구독 및 최신 프레임 처리 | 구현 |
| 후보 안정화 | NMS, 중복 제거 및 다중 프레임 tracking | 구현 |
| 엣지 배포 | VENTUNO Q용 QNN·GenieX·Hexagon NPU 실행 경로 | 장비 환경 검증 필요 |
| 통신 | Wi-Fi TCP 의미 정보 전송 및 결과 bridge | 구현 |
| 시각화 | 웹 대시보드와 IMU 이동 경로 표시 | 구현 |

## 저장소 파일

- `ventuno_wifi_semantic.zip`

  VENTUNO Q와 PC 사이의 Wi-Fi 의미 정보 전송, 웹 대시보드, IMU 연동 및 파이프라인 결과 bridge를 포함합니다.

- `vlm_pipeline_wifi.zip`

  RGB/LWIR 전처리, 4채널 YOLO, VLM 의미 분석, tracking, ROS 2 실시간 센서 입력과 VENTUNO 배포용 C++/Python 파이프라인을 포함합니다.


```bash
git clone https://github.com/SeoYeon-02/D4D.git
cd D4D
git lfs install
git lfs pull
```

