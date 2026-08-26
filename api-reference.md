# API Reference — ai-console

> 표준 5축 문서 ③ 인터페이스 · 최종 확인 **2026-08-26** (소스 라우트 데코레이터 전수)
> **총 54개.** `qa_llm.py` 35 + `aibot_restapi_auth.py` 12 + `aibot_wizard.py` 7.
> 전부 하나의 FastAPI 앱(`api_app`)에 등록된다. 위저드 라우터만 `APIRouter(prefix="/api/wizard")`.
> 기본 접속: `https://<host>:8443`

## 인증 표기

| 표기 | 구현 | 의미 |
|---|---|---|
| **Bearer** | `Depends(get_bearer_api_key_user)` | `Authorization: Bearer <API 키>` |
| **Admin키** | `verify_admin_key` | 관리자 키 |
| **세션** | `Depends(get_current_admin / get_current_master)` | `/api/admin/login` 으로 받은 세션 토큰 |
| **본문키** | 요청 본문의 `api_key` 필드 | 키 소지 자체가 인가. 헤더 인증이 아니다 |
| **없음** | — | 무인증 |

> 외부 연동 시 실제로 쓰는 것은 **Bearer** 4~5개다. 연동 관점 정리는 [docs/api-integration.md](docs/api-integration.md) 를 보라 — 이 문서는 전수 목록이다.

---

## 1. 질의응답 · Agent (10)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| POST | `/api/query/stream` | Bearer | **RAG 채팅 (SSE 스트리밍)** — 연동 1순위 |
| POST | `/agent/chat/completions` | Bearer | **OpenAI 호환** — 기존 클라이언트를 주소만 바꿔 연결 |
| POST | `/agent/chat/completions2` | Bearer | OpenAI 호환 (Direct 경로) |
| POST | `/api/ai/chats` | Bearer | 전체 대화 (세션·저장 포함) |
| POST | `/api/ai/queue` | Bearer | 추가 질의 |
| POST | `/api/ai/summarize` | Bearer | 제목·내용 요약 |
| POST | `/api/ai/validate` | Bearer | 쿼리 검증 및 자동 수정 |
| POST | `/api/ai/playbooks` | Bearer | 플레이북 실행 |
| POST | `/api/ai/chats/regression` | Bearer | 회귀 테스트용 대화 실행 |
| POST | `/api/ai/ticket/memo` | Bearer | 티켓 메모 생성 |

> `config.ini` 에 alias prefix 가 설정되면 `/agent/chat/completions`·`completions2` 에 **접두어 별칭 2개가 런타임 추가**된다(`add_api_route`). 위 54개에는 포함하지 않았다.

## 2. 스트리밍 조각 · 세션 (3)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| GET | `/api/ai/stream/{guid}` | Bearer | 오프셋 기반 응답 조각 반환 |
| DELETE | `/api/ai/stream/{guid}` | Bearer | 응답 캐시 삭제 |
| GET | `/api/ai/hello` | Bearer | 세션 확인 |

## 3. 검색 (3)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| POST | `/api/search` | Bearer | **RAG 검색만** (동기 JSON) — 생성 없이 근거 문서만 |
| GET | `/api/search/status` | Bearer | 검색 시스템 상태 |
| POST | `/api/search/config` | Bearer | 검색 설정 변경 |

## 4. 지식(프롬프트) 관리 (9)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| POST | `/api/ai/prompts/bulk` | Bearer | **YAML 대량 업로드** — 임베딩 + Qdrant 저장. 카트리지 지식 슬롯 교체 통로 |
| GET | `/api/ai/prompts` | Bearer | 목록 조회 |
| POST | `/api/ai/prompts` | Bearer | 단건 생성 |
| DELETE | `/api/ai/prompts` | Bearer | 삭제 |
| GET | `/api/ai/prompts/{guid}` | Bearer | 단건 조회 |
| PUT | `/api/ai/prompts/{guid}` | Bearer | 수정 |
| POST | `/api/ai/prompts/{guid}/{toggle}` | Bearer | 활성/비활성 토글 |
| GET | `/api/ai/sync/manifest` | Bearer | 경량 목록 (싱크용) |
| POST | `/api/ai/sync/points` | Bearer | guid 목록의 벡터+payload 전체 (싱크용) |

## 5. 임베딩 관리 (2)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| GET | `/api/embeddings/list` | Bearer | 임베딩 목록 |
| PUT | `/api/embeddings/{embedding_id}` | Bearer | 수정 및 재임베딩 |

## 6. 온보딩 위저드 (7) — `prefix=/api/wizard`

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| POST | `/api/wizard/prompt-draft` | Bearer | 5문답 → 시스템 프롬프트 초안 (로컬 LLM 생성) |
| POST | `/api/wizard/knowledge-convert` | Bearer | 비정형 문서 → Q&A YAML 초안 (`ingest.py` 가 배치로 호출) |
| POST | `/api/wizard/test-chat` | Bearer | 저장 전 시험 대화 |
| POST | `/api/wizard/cartridge-save` | Bearer | 카트리지 저장 |
| GET | `/api/wizard/cartridges` | Bearer | 카트리지 목록 |
| POST | `/api/wizard/cartridge-mount` | Bearer | 장착 (런타임 반영) |
| POST | `/api/wizard/cartridge-unmount` | Bearer | 해제 |

## 7. 카트리지·운영 (3)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| POST | `/api/cartridge/reload` | **Admin키** | `config.ini [prompts]` 배선을 런타임 재적용 |
| POST | `/api/reload` | **본문키** | 구독의 임베딩 데이터를 메모리에 리로드 |
| POST | `/api/test/complete-incremental` | Bearer | 증분 처리 테스트 |

## 8. 구독·API 키 (8)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| POST | `/api/generate` | **Admin키** | 새 API 키 등록 |
| POST | `/api/list` | **Admin키** | 구독 목록 |
| POST | `/api/verify` | 본문키 | 키 검증 |
| PUT | `/api/update` | 본문키 | 구독 정보 수정 |
| PUT | `/api/renew` | 본문키 | 갱신 |
| DELETE | `/api/delete` | 본문키 | 키 삭제 |
| PATCH | `/api/subscriptions/settings` | 본문키 | 구독 설정 변경 |
| POST | `/api/embedding-status-all` | **Admin키** | 전체 임베딩 상태 |

## 9. 관리자 세션 (4)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| POST | `/api/admin/login` | 없음 | 자격증명 → 세션 토큰 발급 |
| GET | `/api/admin/session` | 세션 | 세션 확인 |
| POST | `/api/admin/change-password` | 세션(master) | 마스터 비밀번호 변경 |
| POST | `/api/admin/logout` | 세션 | 로그아웃 |

## 10. 시스템·페이지 (5)

| Method | Path | 인증 | 설명 |
|---|---|:---:|---|
| GET | `/` | **없음** | 상태 JSON (실측 응답 약 3ms) |
| GET | `/init` | Bearer | 구동 최초 설정 |
| GET | `/api/embedding-status/{sub_id}` | **없음** | 구독별 임베딩 상태 |
| GET | `/wizard` | 없음 | 위저드 SPA (HTML) |
| GET | `/chat` | 없음 | 채팅 UI (HTML) |
| — | `/docs` · `/openapi.json` | **없음** | FastAPI 자동 문서 (기본값 그대로 활성) |

> `/wizard`·`/chat` 은 HTML 만 내려준다. 그 안에서 호출하는 API 는 전부 Bearer 를 요구하므로 페이지 노출 자체로 데이터가 새지는 않는다.

---

## 11. 인증 현황과 한계

**전체 54개 중 Bearer 36개, Admin키 4개, 세션 3개, 본문키 6개, 무인증 5개(+`/docs`).**

주의할 점:

1. **`/docs`·`/openapi.json` 이 무인증으로 열려 있다.** FastAPI 기본값을 그대로 쓰므로 전체 API 스키마가 노출된다. 신뢰할 수 없는 망에 둔다면 `FastAPI(docs_url=None, openapi_url=None)` 로 끄거나 리버스 프록시에서 막아라.
2. **`GET /api/embedding-status/{sub_id}` 가 무인증이다.** `sub_id` 를 바꿔가며 조회할 수 있다.
3. **본문키 6개는 헤더 인증이 아니다.** 요청 본문에 `api_key` 를 실어 보내는 방식이라, 키 소지 자체가 인가가 된다. 프록시·액세스 로그에 본문이 남는 구성이라면 키가 로그에 남는다.
4. **`POST /api/reload` 는 태그가 "관리"인데 관리자 인증이 아니다.** 본문 `api_key` 만 확인한다.
5. **CORS** — `[server] cors_origins` 를 비우면 와일드카드 + 자격증명 차단(기본, 안전), 지정하면 그 오리진만 + 자격증명 허용이다. 와일드카드와 자격증명을 동시에 켜지 않도록 코드에서 막아 두었다. 상세는 [security-review.md](security-review.md) S-4.
6. **콘솔은 기본적으로 `0.0.0.0` 에 바인드되고 Qdrant 는 인증 없이 뜬다.** 위 항목들은 전부 이 전제 위에서 위험이 커진다. 방화벽으로 포트를 막아라.

## 12. 버전 안정성

[docs/api-integration.md](docs/api-integration.md) 가 **유지 대상**으로 명시한 것은 다음 4개 경로다. 나머지 내부 엔드포인트는 예고 없이 바뀔 수 있다.

`POST /api/query/stream` · `POST /agent/chat/completions` · `POST /api/search` · `POST /api/ai/prompts/bulk`

## 13. CLI

| 명령 | 설명 |
|---|---|
| `./install.sh` | HW 감지 → 티어 판정 → 프리셋 → 빌드 → `config.ini` 생성. `--preset` · `--tier` · `--yes` · `--dry-run` · `--no-model` |
| `./run.sh {start\|stop\|restart}` | 콘솔 기동·정지·재기동 |
| `aibotctl cartridge {validate\|mount\|unmount\|purge}` | 카트리지 수명주기 |
| `python ingest.py <디렉터리> <카트리지명>` | 문서 → 지식 카트리지 배치 변환 |

## 14. 검증 방법

```bash
# 라우트 전수 재확인 (문서와 코드가 어긋났는지)
grep -c '@api_app\.\(get\|post\|put\|delete\|patch\)("' qa_llm.py              # 35
grep -c '@app\.\(get\|post\|put\|delete\|patch\)("'     aibot_restapi_auth.py  # 12
grep -c '@router\.\(get\|post\|put\|delete\|patch\)("'  aibot_wizard.py        # 7

# 기동 중이면 스키마로 직접
curl -sk https://localhost:8443/openapi.json | jq '.paths | keys | length'
```
