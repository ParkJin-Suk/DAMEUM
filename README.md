# 담음 (DAMEUM)

> 부모와 감정과 목소리를 담음

구음장애(마비말장애)가 있는 부모의 발화를 텍스트로 복원하고, 감정을 반영한 또렷한 **부모 본인의 음색**으로 다시 합성해 아이에게 동화책과 자장가를 들려줄 수 있게 돕는 모바일 앱입니다.

**2026 하나-SKT 해커톤** 출품작 · Tech4Good

https://www.youtube.com/watch?v=iuZVZYoQftM

## 문제 정의

발음이 불분명해지는 구음장애가 있는 부모는 아이에게 책을 읽어주거나 자장가를 불러주기 어렵습니다. 담음은 부모가 말한 그대로를 인식·교정하고, 부모의 목소리 특성과 문장의 감정을 살려 아이가 알아듣기 쉬운 음성으로 돌려줍니다.

## 주요 기능

- **목소리 등록**: 부모의 목소리 프로필을 만들고 음성을 등록
- **새 동화책 담음**: 동화책 페이지를 녹음하면 발화를 복원해 부모 음색의 낭독 음성으로 생성
- **자장가 담음**: 자장가 가사를 부드럽게 낭독하거나, 무반주 원곡의 음높이와 리듬을 유지한 채 부모 음색으로 가창 변환
- **라이브러리 / 대시보드**: 최근 읽은 책과 들은 자장가 관리

## AI 파이프라인

생성 요청은 `202 Accepted`로 접수된 뒤 단일 큐에서 순차 처리되며, 프론트엔드는 작업 상태를 폴링합니다.

```
녹음 → STT → 문장 복원 → 감정 분류 → 음성 합성(TTS) → 재생
```

| 단계 | 모델/런타임 |
|---|---|
| STT | Whisper small + 구음장애 LoRA (CPU FP32) |
| 문장 복원 | Qwen3-1.7B GGUF Q8_0 + llama.cpp |
| 감정 분류 | KoELECTRA 한국어 6감정 분류 |
| 음성 합성 | VoxCPM2 2B |
| 가창 음색 변환 | Seed-VC 44.1kHz (CPU FP32) |

### STT 모델 (직접 미세조정)

AI-Hub 한국어 구음장애 음성 데이터(뇌신경장애 subset)로 Whisper small에 LoRA를 학습했습니다.

- held-out 125개 음성 기준 **CER 64.3% → 53.6%**
- 학습 데이터 약 24분 분량의 시연용 모델이며, 의료·안전 관련 문장의 정확성은 보장하지 않습니다.

### 로컬 실행 설계

개인 음성 데이터를 외부로 보내지 않도록 전체 파이프라인을 **로컬 CPU(RAM 16GB)** 에서 실행하도록 설계했습니다. 메모리 경합을 줄이기 위해 작업이 끝나면 모델을 해제하고(Qwen은 idle 시 자동 sleep), 모델을 순차 처리합니다. 세부 실측 수치는 [`dameum-fastapi/README.md`](dameum-fastapi/README.md)를 참고하세요.

## 기술 스택

| 영역 | 스택 |
|---|---|
| Frontend | React Native 0.81, Expo SDK 54, React 19, React Navigation |
| Backend | FastAPI, SQLAlchemy(async) + SQLite, Uvicorn |
| AI | Whisper + LoRA(PEFT), Qwen3 / llama.cpp, KoELECTRA, VoxCPM2, Seed-VC |
| Docs | OpenAPI(Swagger) 기반 프론트 연동 명세 (`dameum-fastapi/docs/API.md`) |

## 프로젝트 구조

```
.
├── src/                 # 앱 화면, 컴포넌트, API 클라이언트, 상태(Context)
│   ├── screens/         # 인트로, 프로필, 목소리 등록, 동화책, 자장가, 라이브러리 등
│   ├── api/client.js
│   └── store/
└── dameum-fastapi/      # 로컬 AI 파이프라인 REST API
    ├── app/             # API, 작업 큐, 추론, 가창 변환
    ├── artifacts/stt/   # 구음장애 Whisper LoRA 어댑터
    ├── assets/lullabies/# 무반주 자장가 프리셋
    ├── docs/            # API 명세, openapi.json
    └── tests/
```

## 실행 방법

### 1. 백엔드

자세한 설치(llama.cpp, Seed-VC 포함)는 [`dameum-fastapi/README.md`](dameum-fastapi/README.md)를 따릅니다.

```bash
cd dameum-fastapi
uv sync --python 3.11 --extra ml --extra dev
cp .env.example .env          # DAMEUM_API_KEY 설정
uv run uvicorn app.main:create_app --factory --host 127.0.0.1 --port 8000 --workers 1
```

Swagger UI: http://127.0.0.1:8000/docs

### 2. 프론트엔드

```bash
npm install
npx expo start        # --ios / --android / --web
```

## 팀 구성 및 역할

| 조원 | 역할 |
|---|---|
| 고시은 | 기획 / ux/ui |
| 구준모 | PM / ai |
| 김성희 | ai |
| 박진석 | 기획 / ppt 제작 / 발표 |
| 이백범 | 프론트 |
| 이인명 | 백엔드 |
| 장연재 | 기획 |
| 최성현 | 백엔드 |

## 성과

https://app.notion.com/p/3a0e57fad6df80109178d204666f40b1

## 참고

- 원본 저장소: [HaemulPajeon/DAMEUM](https://github.com/HaemulPajeon/DAMEUM)
- 자장가 프리셋 출처와 라이선스: `dameum-fastapi/assets/lullabies/README.md`
- Seed-VC는 GPL-3.0이므로 FastAPI 의존성과 분리된 격리 환경에서 실행합니다.
