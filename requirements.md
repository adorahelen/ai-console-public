# Requirements — ai-console

> 표준 5축 문서 ① 요구사항 · 최종 확인 **2026-08-26**

## 1. 목적 & 대상 사용자

**나만의 온프레미스 AI 에이전트 본체.** 프로덕션 SecOps 에이전트에서 도메인을 걷어낸 범용 콘솔로, 엔진(RAG · intent · 멀티백엔드 서빙)은 고정하고 도메인(프롬프트) · 지식(RAG 문서) · 모델(HW 맞춤)을 **카트리지 3슬롯**으로 교체한다.

**대상 사용자**
- 사내 문서·매뉴얼을 근거로 답하는 에이전트를 **자기 하드웨어에서** 굴리려는 개인·소팀
- 외부 LLM API 로 데이터를 보낼 수 없는 환경(망분리·반출 제한)
- 도메인마다 앱을 새로 짓지 않고 **하나의 엔진에 카트리지만 갈아끼우려는** 운영자

**명시적 비목표**
- 파인튜닝. 도메인 능력은 순정 가중치 위의 프롬프트 + RAG 로만 만든다
- LangChain 류 오케스트레이션 레이어 도입
- 멀티테넌시·클러스터·수평 확장

## 2. 기능 요구사항

### 2.1 질의 파이프라인
질문 → **Bearer 인증** → **intent 분류**(카트리지 프롬프트가 축을 정의) → **PII 마스킹**(선택) → **RAG 검색**(BGE-M3 dense + colbert 2-way → RRF) → intent 프롬프트 + context 주입 → **스트리밍 응답**.

### 2.2 카트리지 3슬롯
| 슬롯 | 교체 통로 | 코드 수정 |
|---|---|:--:|
| 프롬프트 | `config.ini [prompts]` 경로 외부화 | 불필요 |
| 지식 | `POST /api/ai/prompts/bulk` → 임베딩 → Qdrant | 불필요 |
| 모델 | `config.ini [model]` 한 줄 + `models.yaml` 프리셋 | 불필요 |

장착(`aibotctl cartridge mount` / 위저드)은 **콘솔 재시작 없이 런타임 반영**된다. 미반영 시 `./run.sh restart` 가 폴백.

### 2.3 지식 구축 2경로
- **위저드**(`/wizard`) — 캐릭터 서술 + 지식 붙여넣기 → 설치된 LLM 이 내부 포맷으로 변환(자기부트스트랩) → 카트리지 저장
- **배치 인제스트**(`ingest.py`) — 디렉터리 단위 추출 → 변환 → validate → 카트리지 생성
  - 구조화 소스(`question`/`answer` 열이 있는 csv·json·xlsx)는 **결정적 직접 매핑(무손실)**
  - 비정형 문서는 로컬 LLM 이 Q&A **초안** 생성 → **검수 필수**. `validate` 는 stub 만 거르고 정답 여부는 검증하지 못한다
  - PDF·DOCX·XLSX·이미지(OCR)는 추출 라이브러리 설치 시 처리, 미설치면 그 파일만 건너뛰고 안내
- 콘솔이 먹는 지식 포맷은 **Q&A YAML 하나**. 포맷 다양성은 입력단에서 흡수한다

### 2.4 멀티 백엔드 서빙
핸들러 6종(llama · gemma · gpt_oss · openai · claude · qwen)을 레지스트리로 매핑한다. 로컬 `llama-server`(GGUF)와 외부 API(OpenAI · Claude)를 동일 인터페이스로 처리하며 전환은 설정 한 줄이다.

### 2.5 설치기
HW 감지(VRAM · RAM) → 티어 판정 → `models.yaml` 프리셋 제안 → 시스템 의존성 → venv → **llama.cpp 빌드**(릴리스 태그 고정, GPU 면 CUDA) → Qdrant 바이너리 → 모델 다운로드 → `config.ini` 생성.

프리셋 선별은 **2단계**다. ① VRAM 으로 티어를 정하고 ② 그 티어 안에서 `min_vram_gb` 와 `min_ram_gb` 를 **둘 다** 넘는 후보만 남긴 뒤 실측 tok/s 최고값을 기본으로 쓴다. VRAM 과 RAM 은 대체재가 아니라 AND 조건이다. **후보가 0건이면 설치를 강행하지 않고** 사유와 우회 경로를 안내한 뒤 멈춘다.

옵션: `--preset` · `--tier` · `--yes`(비대화형) · `--dry-run`(계획만) · `--no-model`.

### 2.6 외부 연동
[docs/api-integration.md](docs/api-integration.md) 의 3패턴 — 임베드형(`/api/query/stream`) · 백엔드 호출형(`/api/search`) · 데이터 적재형(`/api/ai/prompts/bulk`). OpenAI 호환 경로(`/agent/chat/completions`)로 기존 LLM 클라이언트를 주소 변경만으로 붙일 수 있다. 전체 목록은 [api-reference.md](api-reference.md) (54개).

### 2.7 운영
`run.sh` 기동·재기동, `ai-agent.service` systemd 등록, `docker/` 컨테이너 배포, `aibotctl` 카트리지 CLI, 한 호스트 다중 인스턴스([docs/multi-instance.md](docs/multi-instance.md)).

## 3. 비기능 요구사항 / 제약

### 3.1 하드웨어
| 티어 | 조건 | 기본 프리셋 | 실측 |
|---|---|---|---|
| `cpu-only` | GPU 없음 | RAM 8GB↑ 에서 e2b/e4b | ⚠️ **미측정** |
| `gpu-8gb` | VRAM 7↑ | e4b-q4 | ⚠️ **미측정** |
| `gpu-16gb` | VRAM 12↑ | 12b-q4 | 7,836MiB · 99.1 tok/s |
| `gpu-24gb+` | VRAM 24↑ | 26b-full | 221 tok/s · TTFT 16ms |
| `api` | `--tier api` 명시 | 외부 API | HW 무관 |

- `VRAM_GB` 는 `nvidia-smi` 보고값 ÷ 1024 의 **정수 몫**이다. 24GB 카드도 보고값이 24,576MiB 미만이면 23 → `gpu-16gb` 로 떨어진다
- `api` 티어는 자동 감지로 도달하지 않는다
- 임베딩(BGE-M3)이 전 티어 공통으로 **VRAM 약 1.0GB** 를 더 쓴다. 위 요건에 포함돼 있다
- RAM 4GB 는 요건 미달로 설치 중단

### 3.2 성능 특성
매 요청마다 긴 시스템 프롬프트 + RAG 컨텍스트가 prefix 로 깔리는 구조다. 전 티어를 `runtime=server` 로 통일해 KV 캐시 재사용(`--cache-reuse`)이 기본으로 붙도록 했고, 그 효과가 큰 구조다.

### 3.3 보안
- 인증: Bearer API 키가 기본(54개 중 36개). 관리 계열은 Admin 키 또는 세션 토큰
- **PII 마스킹은 `[pii] pii_mode = True` 일 때만 동작하며 기본 off.** 적용 범위가 핸들러마다 다르다 — 외부 API 티어(openai · claude)와 gemma 는 전 경로, 나머지 로컬 핸들러는 `completions2` 경로만. 개인정보 보호가 도입 요건이면 이 차이를 먼저 확인할 것 ([security-review.md](security-review.md) S-6)
- **콘솔은 기본적으로 `0.0.0.0` 에 바인드되고 Qdrant 는 인증 없이 뜬다.** 신뢰할 수 없는 망에 두지 말고 방화벽으로 막을 것
- `/docs` · `/openapi.json` 이 무인증으로 열려 있다(FastAPI 기본값)
- CORS 는 `cors_origins` 를 비우면 와일드카드 + 자격증명 차단, 지정하면 그 오리진만 + 자격증명 허용. 와일드카드와 자격증명 동시 허용은 코드에서 막았다 (S-4)
- 외부 API 백엔드를 고르면 **데이터가 외부로 나간다.** 온프레미스가 요건이면 로컬 핸들러로 고정할 것

### 3.4 운영 제약
- 단일 프로세스. 이중화·클러스터 없음. 다중 인스턴스는 부하 분산이 아니라 **용도 분리**용
- 파인튜닝을 하지 않으므로 지식 갱신은 재학습이 아니라 **재적재**다
- llama.cpp 는 릴리스 태그를 고정해 빌드한다(상류 변경이 그대로 전파되지 않도록)

### 3.5 현재 상태 — pre-release
| 영역 | 상태 |
|---|---|
| 엔진 (RAG · intent · 핸들러 게이트웨이) | 동작 |
| `install.sh` | 동작 (`--dry-run` 으로 미리 확인 가능) |
| 웹 온보딩 위저드 · 채팅 UI | 동작 |
| 카트리지 CLI (validate · mount · unmount · purge) | 동작 |
| **깨끗한 환경 설치 종단 검증** | **미완** |
| 다중 GPU 분리 배정 | 미검증 |
| 저사양 4개 티어(`cpu-only` · `gpu-8gb`) 실측 | 미측정 — `min_ram_gb` 8·12 는 보수적 추정치 |

## 4. 범위 외

- 파인튜닝 · 학습 파이프라인
- 멀티테넌시 · 클러스터 · 수평 확장 · 부하 분산
- 오케스트레이션 프레임워크(LangChain 류) 도입
- 지식 정확성의 자동 검증 — `validate` 는 형식만 본다. 내용 검수는 사람 몫이다
- 상용 지원 · SLA

## 5. 완료 정의

**깨끗한 리눅스 호스트에서 `./install.sh` 한 줄로 설치가 끝나고, 위저드에서 만든 카트리지가 재시작 없이 물려 자기 문서로 답한다.** 여기까지가 pre-release 해제 조건이며, 현재 남은 것은 §3.5 의 "설치 종단 검증"이다.
