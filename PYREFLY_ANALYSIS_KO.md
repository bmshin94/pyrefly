# Pyrefly 전수조사 및 활용 분석 (한국어 정리)

> 이 문서는 Pyrefly 저장소를 전수조사한 결과와, 이를 어떻게 쓰고 무엇을 만들 수 있는지에 대한
> 분석을 한국어로 정리한 것이다. 코드 변경은 없고, 문서만 추가한다.

## 관련 링크

| 구분 | 주소 |
| --- | --- |
| 이 저장소 (포크) | https://github.com/bmshin94/pyrefly |
| 원본 저장소 (Meta 공식) | https://github.com/facebook/pyrefly |
| 공식 웹사이트 / 문서 | https://pyrefly.org · https://pyrefly.org/en/docs/ |
| 브라우저 샌드박스 (WASM) | https://pyrefly.org/sandbox/ |
| PyPI 패키지 | https://pypi.org/project/pyrefly/ |
| VS Code 확장 | https://marketplace.visualstudio.com/items?itemName=meta.pyrefly |
| Open VSX 확장 | https://open-vsx.org/extension/meta/pyrefly |
| Zed 확장 | https://zed.dev/extensions/pyrefly |
| 커뮤니티 (Discord) | https://discord.gg/Cf7mFQtW7W |
| 기여 가이드 | https://github.com/facebook/pyrefly/blob/main/CONTRIBUTING.md |
| 아키텍처 개요 | https://github.com/facebook/pyrefly/blob/main/ARCHITECTURE.md |
| 참고: typeshed | https://github.com/python/typeshed |

---

## 1. 이 저장소의 정체

Pyrefly는 Meta가 Rust로 개발한 **Python 타입 체커 + 언어 서버(LSP)** 이다.
이 저장소는 원본 `facebook/pyrefly`를 `bmshin94/pyrefly`로 포크한 사본이며,
포크 측에서 추가된 것은 AI 협업용 `CLAUDE.md` 페르소나 가이드 문서다.

| 항목 | 값 |
| --- | --- |
| 구현 언어 | Rust (Rust 파일 707개, 약 495,000줄) |
| 라이선스 | MIT (상업적 이용 가능) |
| 개발 상태 | 1.0.0 이후 stable, 매월 마이너 릴리스 |
| 빌드 시스템 | cargo (오픈소스) / Buck (Meta 내부) |
| 배포 경로 | PyPI, VS Code Marketplace, Open VSX |

### 핵심 기능

1. **타입 체커** — 코드를 실행하지 않고 정적으로 타입 오류를 검출한다.
2. **언어 서버(LSP)** — 자동완성, 호버 타입 정보, 정의로 이동, 인레이 힌트,
   시맨틱 하이라이팅 등 IDE 기능의 엔진 역할을 한다.
3. **성능** — 초당 약 185만 줄 검사, Mypy/Pyright 대비 약 15배 빠름,
   저장 후 재검사 10ms 이내. Instagram의 2천만 줄 코드베이스에서 기본 타입 체커로
   운영되며 PyTorch, JAX도 채택했다.
4. **텐서 셰이프 추적** — PyTorch/NumPy/JAX/einops 텐서의 shape를 타입 수준에서 검증한다.
   학습을 실행하기 전에 shape 불일치를 잡는다. 이것이 다른 타입 체커와의 가장 큰 차별점이다.

### 동작 원리 (3단계)

`ARCHITECTURE.md`에 기술된 파이프라인은 다음과 같다.

1. **exports 파악** — 각 모듈이 무엇을 내보내는지 결정한다. `import *`를 전이적으로 해결한다.
2. **bindings 생성** — 모듈 단위로 문장과 스코프 정보를 중간 표현으로 변환한다.
3. **bindings 해결** — 바인딩을 풀어 타입을 확정한다. 재귀 등으로 값을 알 수 없을 때는
   `Type::Var` 자리표시자를 삽입하고 나중에 채운다.

모듈 단위 증분 검사와 병렬성에 최적화되어 있으며, 심볼 단위 지연 해결(Salsa 방식)은 쓰지 않는다.

---

## 2. 폴더 구조 전수조사

### Rust 본체

| 경로 | 역할 |
| --- | --- |
| `pyrefly/lib/export` | 모듈 export 파악 (1단계) |
| `pyrefly/lib/binding` | 바인딩 생성 (2단계) |
| `pyrefly/lib/alt` | 바인딩 해결 (3단계) |
| `pyrefly/lib/solver` | 타입 변수 해결, 할당 가능성 판정 |
| `pyrefly/lib/lsp` | 언어 서버 프로토콜 구현 |
| `pyrefly/lib/tsp` | TSP(Type Server Protocol) 서버 |
| `pyrefly/lib/commands` | CLI 명령어 전체 |
| `pyrefly/lib/error` | 에러 수집·출력, 에러 종류별 심각도 |
| `pyrefly/lib/module` | import 해석, 모듈 탐색 |
| `pyrefly/lib/state` | 언어 서버 내부 상태, 증분 재검사 |
| `pyrefly/lib/stubgen` | `.py` → `.pyi` 스텁 생성 |
| `pyrefly/lib/query`, `report` | 쿼리 API, 커버리지 리포트 |
| `pyrefly/lib/test` | 타입 체커 통합 테스트 |

### 크레이트 (`crates/`)

| 크레이트 | 역할 |
| --- | --- |
| `pyrefly_types` | 내부 타입 표현. `dimension.rs`, `einops.rs`, `einsum.rs`, `data_frame.rs` 포함 |
| `pyrefly_config` | 설정 포맷, mypy/pyright 설정 마이그레이션 |
| `pyrefly_python` | Python 언어 모델링 (모듈, `sys.version_info` 등) |
| `pyrefly_bundled` | typeshed 표준 스텁 번들 |
| `pyrefly_util` | 범용 유틸 (IO, 락, 스레드풀) |
| `pyrefly_graph` | 값 인덱싱, 상호 의존 계산 캐싱 |
| `pyrefly_derive` | `TypeEq`, `Visit` 파생 proc-macro |
| `pyrefly_build` | Buck/Bazel 빌드 연동 |
| `pyrefly_lsp_test` | LSP 테스트 하네스 |
| `pyrefly_bench_harness` | 벤치마크 하네스 |
| `tsp_types` | TSP 프로토콜 타입 |
| `pyrefly_glean_schema` | Meta Glean 코드 인덱싱 스키마 |

### 그 외 디렉터리

| 경로 | 내용 |
| --- | --- |
| `website/` | pyrefly.org 소스 (Docusaurus). 설치·설정·IDE·Django·Pydantic·pytest·텐서 셰이프 문서 |
| `lsp/` | VS Code 확장 (TypeScript, publisher `meta`) |
| `tensor-shapes/` | PyTorch/NumPy/JAX/einops shape 스텁과 테스트 |
| `pyrefly_wasm/` | WASM 빌드. 브라우저 샌드박스의 기반 |
| `conformance/` | python/typing 표준 적합성 테스트 (수동 편집 금지) |
| `scripts/` | 벤치마크, 타입 체커 비교, 릴리스 노트 생성, mypy_primer 래퍼, LLM 기반 이슈 랭킹·분류 |
| `test/` | 마크다운으로 작성된 CLI/IDE E2E 테스트 |
| `release_notes/` | v0.58 ~ v1.2 릴리스 노트와 생성 프롬프트 |
| `schemas/` | `pyrefly.toml` JSON 스키마 |
| `action.yml` | GitHub Action. PR에 타입 오류를 인라인 주석으로 표시 |
| `.github/workflows/` | CI 34개 (바이너리·확장·웹사이트 배포, 이슈 자동 분류 등) |
| `test.py` | 린트와 테스트를 함께 실행하는 통합 러너 |

### AI 협업 관련 파일

| 경로 | 내용 |
| --- | --- |
| `AGENTS.md` | AI 에이전트 작업 규칙. 코딩 스타일(KISS/DRY), 커밋 메시지, 테스트 작성법, 환경 감지 규칙 |
| `CLAUDE.md` | 이 포크에서 추가한 페르소나 가이드 |
| `.claude/CLAUDE.md` | `AGENTS.md`를 임포트하는 연결 파일 |
| `REVIEW.md` | 코드 리뷰 기준 |
| `.agents/skills/` | Claude Skill 4개 (diff 리뷰, 벤치마크, shape DSL 수정, torch shape 예제 추가) |
| `tensor-shapes/skills/` | Claude Skill 1개 (torch 모델에 shape 타입 추가) |

---

## 3. 설치 및 사용법

### 도구로 사용 (소스 빌드 불필요)

```bash
pip install pyrefly            # 또는 uv / poetry / conda / pixi
cd 프로젝트_디렉터리
pyrefly init                   # 설정 생성 + mypy/pyright 설정 마이그레이션
pyrefly check --summarize-errors
pyrefly suppress               # 기존 에러를 일괄 무시 처리하고 깨끗하게 시작
```

`uv`를 쓰면 설치 없이 바로 실행할 수 있다: `uvx pyrefly check`

### CLI 명령어

| 명령어 | 역할 |
| --- | --- |
| `check` | 프로젝트/파일 전체 타입 검사 |
| `snippet` | 코드 조각 검사 |
| `init` | 설정 생성 및 기존 타입 체커 설정 마이그레이션 |
| `suppress` | 무시 주석 삽입 / 불필요한 무시 제거 |
| `infer` | 타입 주석 자동 추가 |
| `stubgen` | `.pyi` 스텁 생성 |
| `coverage` | 타입 커버리지 리포트 |
| `lsp` / `tsp` | 언어 서버 / TSP 서버 실행 |
| `dump-config` | 현재 적용 중인 설정 출력 |
| `buck-check` / `bazel-check` | Buck / Bazel 빌드 연동 |

### 에디터

- VS Code 계열: 마켓플레이스에서 `Pyrefly`(publisher `meta`) 설치 후 Python 파일을 열면 자동 활성화된다.
- Neovim, Zed, Emacs, Vim: `pyrefly lsp`를 LSP 서버로 등록한다.
- 확장은 기본적으로 활성 Python 환경에 설치된 Pyrefly를 찾아 실행하고, 없으면 번들 바이너리로 대체한다.
- 기본값은 basic 프리셋(확신도 높은 오류만)이다. 전체 진단을 원하면 `pyrefly init`을 실행하거나
  에디터 설정에서 `python.pyrefly.typeCheckingMode`를 `default` 또는 `strict`로 바꾼다.

### 설정 파일

`pyrefly.toml` 또는 `pyproject.toml`의 `[tool.pyrefly]` 섹션을 사용한다.

```toml
[tool.pyrefly]
project-includes = ["src"]
project-excludes = ["**/node_modules", "build"]
python-version = "3.12"
check-unannotated-defs = true

[tool.pyrefly.errors]
bad-assignment = "error"
missing-attribute = "warn"
```

설정이 없으면 근처의 mypy/pyright 설정을 메모리상에서 마이그레이션해 쓰고,
그것도 없으면 basic 프리셋을 사용한다.

### CI 연동

```yaml
- uses: facebook/pyrefly@main
  with:
    python-version: "3.12"
    args: "--summarize-errors"
```

내부적으로 `pyrefly check --output-format=full-text-with-github`를 실행해 PR에 인라인 주석을 남긴다.

### 저장소 자체를 개발할 때

```bash
cargo build --release
cargo test <테스트_이름>

# 커밋 전 필수 (AGENTS.md 규칙)
python3 test.py --no-test --no-tensor-shapes --no-conformance --no-jsonschema --no-extension

# 전체 테스트 (무거움)
python3 test.py
```

---

## 4. 분류: 플러그인 / 스킬 / MCP

Pyrefly 본체는 **독립 실행 네이티브 바이너리이며 동시에 LSP 서버**다.

| 분류 | 해당 여부 | 설명 |
| --- | --- | --- |
| 독립 CLI 도구 | 해당 | `pyrefly` 실행 파일 |
| LSP 서버 | 해당 | Language Server Protocol 구현 |
| VS Code 확장 | 부분 해당 | `lsp/`는 얇은 래퍼이고 본체는 Rust 바이너리 |
| Claude Skill | 본체는 아님 | 단, `.agents/skills/`에 기여자용 스킬 5개가 있다 |
| MCP 서버 | 해당 없음 | 저장소 전체 검색 결과 MCP 구현이 없다 |

LSP(Language Server Protocol, Microsoft)와 MCP(Model Context Protocol, Anthropic)는
이름이 비슷하지만 목적이 다르다. LSP는 에디터와 언어 서버 사이의 프로토콜이고,
MCP는 모델과 외부 도구 사이의 프로토콜이다. 둘 다 JSON-RPC 기반이므로
Pyrefly를 감싸는 MCP 서버를 만드는 것은 자연스러운 확장 방향이다.

## 5. API 토큰

Pyrefly 사용에는 API 토큰, 로그인, 요금이 전혀 필요하지 않다. 완전히 로컬에서 동작하며
소스 코드가 외부로 전송되지 않는다.

토큰이 필요한 경우는 Pyrefly 사용과 무관한 유지보수 작업뿐이다.

- `scripts/issue_ranker`, `scripts/primer_classifier` 등 LLM 기반 메인테이너 자동화 스크립트
- GitHub Actions 실행 (`GITHUB_TOKEN`은 GitHub가 자동 발급)
- PyPI / VS Code 마켓플레이스 배포 (Meta 공식 릴리스)

## 6. GitHub에서 주목받는 이유

1. Meta 공식 프로젝트라는 신뢰도.
2. Mypy/Pyright 대비 약 15배라는 측정 가능한 성능 우위.
3. ruff, uv로 이어지는 "Python 도구를 Rust로 재작성" 흐름의 연장선.
4. OCaml로 작성된 Pyre를 Rust로 다시 쓴 계보, 그리고 Astral의 `ty`와의 경쟁 구도.
5. Instagram 2천만 줄, PyTorch, JAX라는 대규모 실전 검증 사례.
6. 이주 장벽을 낮춘 설계: `pyrefly init`의 설정 자동 마이그레이션, `pyrefly suppress`의
   기존 오류 일괄 침묵, 기본 basic 프리셋으로 노이즈 최소화. 한 파일부터 점진적으로 도입할 수 있다.
7. 메이저 타입 체커 중 유일한 텐서 shape 정적 검증 기능.
8. 공식 문서, 브라우저 샌드박스, IDE 확장, GitHub Action, 프레임워크별 지원 문서까지 갖춘 완성된 생태계.

## 7. 로컬 에이전트 구축에 주는 가치

### 에이전트의 검증 도구로 활용

LLM이 생성한 코드를 `pyrefly check`로 검증하고, 오류 메시지를 다시 모델에 피드백해
수정 루프를 돌리는 구조를 만들 수 있다. Pyrefly가 에이전트 루프에 적합한 이유는 다음과 같다.

- 재검사가 10ms 수준이라 루프를 여러 번 돌려도 지연이 체감되지 않는다.
- 완전 로컬 실행이라 API 비용과 코드 유출 위험이 없다.
- `--output-format`으로 JSON/SARIF 출력을 얻어 모델 입력으로 바로 쓸 수 있다.
- `pyrefly snippet`으로 파일을 만들지 않고 코드 조각만 검증할 수 있다.
- LSP 모드를 통해 정의 위치, 심볼 타입, 호버 정보를 질의하면 grep보다 정확하고
  토큰 사용량도 적게 코드베이스를 파악할 수 있다.
- 텐서 shape 검증까지 함께 얻을 수 있다.

### 에이전트 설계 참고 자료로 활용

- `AGENTS.md`: `BUCK` 파일 존재 여부로 환경을 스스로 판별하게 하는 규칙, 코딩 철학의 명문화,
  "도달 불가능한 상태를 조용한 폴백으로 숨기지 말라"는 제약, 커밋 메시지 규칙,
  `bug = "..."` 마커로 알려진 결함을 통과하는 테스트로 문서화하는 기법.
- `.agents/skills/`: YAML 프론트매터와 마크다운 본문으로 구성된 스킬 포맷, 그리고
  여러 대상을 병렬 서브에이전트로 처리하라는 오케스트레이션 지시.
- `scripts/`: `llm_transport.py`의 LLM 호출 추상화, `issue_ranker`와 `primer_classifier`의
  LLM 기반 분류 파이프라인, `generate_release_notes.py`와 `release_notes/prompt.md`처럼
  프롬프트를 코드와 분리해 관리하는 방식.

### 직접 만들 수 있는 것: Pyrefly MCP 서버

Pyrefly CLI 또는 LSP를 감싸 MCP 도구로 노출한다. 예: `check_python`, `check_snippet`,
`get_type_at`, `goto_definition`, `infer_types`, `check_tensor_shapes`.
구현 난이도는 낮고, 모든 AI 코딩 도구가 Python 타입 검증을 필요로 하므로 활용 가치가 높다.

## 8. React / PHP로 만들 수 있는가

Pyrefly 자체를 React나 PHP로 재구현하는 것은 권하지 않는다. 핵심 가치가 성능이고,
대규모 병렬 처리와 메모리 제어가 필요하기 때문이다. PHP는 요청 단위 실행 모델이라
장시간 유지되는 언어 서버와 맞지 않는다.

반면 Pyrefly를 엔진으로 두고 그 위에 제품을 올리는 것은 매우 적합하다.

### React로 만들 수 있는 것

- `pyrefly_wasm`을 사용한 브라우저 내 타입 검사. Monaco Editor와 결합하면 서버 비용 없이
  동작하는 웹 플레이그라운드가 된다.
- `pyrefly coverage` JSON을 받아 렌더링하는 타입 커버리지 대시보드.
- JSON/SARIF 출력을 필터링·그룹핑하는 대량 오류 탐색 UI.
- 신경망 레이어별 텐서 shape 흐름 시각화 도구.

### PHP로 만들 수 있는 것

웹 백엔드 및 오케스트레이션 계층에 적합하다. Pyrefly CLI를 호출해 JSON 결과를 받고,
검사 이력을 저장하고, GitHub 웹훅을 받아 검사를 트리거하고, 조직·과금을 관리한다.
사용자 코드를 서버에서 실행하는 형태라면 명령 주입 방지와 샌드박스가 필수다.

권장 구조는 React 프론트엔드 + (WASM 즉시 검사 / 백엔드 API 무거운 검사) + Pyrefly CLI 엔진이다.

## 9. 수익화 아이디어

MIT 라이선스이므로 상업적 이용이 자유롭다. 저작권 고지와 라이선스 사본을 유지해야 하고,
"Pyrefly"와 "Meta" 상표를 제품명으로 쓰지 않아야 한다. "Powered by Pyrefly" 같은 표기는 가능하다.
`pyrefly_bundled`의 typeshed 스텁은 별도 라이선스를 따르므로 재배포 시 확인이 필요하다.

### 1) 텐서 셰이프 검증 서비스

다른 타입 체커가 제공하지 않는 기능이라 차별성이 가장 크다. 해결하는 고통이 명확하다:
학습을 수십 분 돌린 뒤에야 발생하는 shape 불일치 오류는 GPU 비용과 실험 사이클 지연으로 직결된다.

- 1단계: pre-commit 훅 형태의 오픈소스 도구로 신뢰를 확보한다.
- 2단계: GitHub App으로 PR마다 shape 검증, shape 흐름 시각화, shape 회귀 추적을 제공한다.
- 3단계: 사내 모델 라이브러리 shape 감사, 커스텀 레이어용 `.pyi` 스텁 제작 대행, CI 정책 강제.
- 가격: 팀 단위 구독 + 엔터프라이즈 연간 계약 + 감사 프로젝트 단건.
- 대상: AI 스타트업, 대기업 ML 플랫폼팀, 자율주행·의료영상 등 텐서 중심 도메인.

### 2) AI 코드 검증 API

LLM이 생성하는 Python 코드의 양은 늘고 있으나 검증 계층은 부족하다.

- 무료 MCP 서버로 포지션을 확보하고, 유료 검증 API로 수익화한다.
- 검사 단가가 사실상 0에 가까워 마진이 크다.
- 오류를 모델에 피드백해 통과까지 반복하는 자동 수정 루프는 고부가 상품이 된다.
- 리스크는 대형 업체의 유사 제품이며, shape 검증과 자동 수정 루프로 차별화한다.

### 3) 마이그레이션 컨설팅

가장 빠르게 현금화할 수 있다. 1.0 stable 출시와 대형 프로젝트 채택으로 이주 수요가 늘고 있지만,
대규모 레거시 코드베이스의 이주는 외부 전문가를 필요로 한다.

- 진단(커버리지 측정, 오류 분류, 로드맵) → 실행(설정 적용, CI 통합, 핵심 모듈 주석화, 팀 교육)
  → 유지(월간 리포트, 오류 예산 관리) 패키지로 구성한다.
- `crates/pyrefly_config/migration`의 변환 로직까지 파악하고 있으면 설정 매핑을 정확히 설명할 수 있다.
- 영업 경로: 이주 후기 기술 블로그, mypy 사용 대형 저장소에 무료 진단 리포트 발송, 커뮤니티 기여, 컨퍼런스 발표.

### 4) 교육 콘텐츠

- "Rust로 타입 체커 만들기" 강의/책 — 이 저장소가 실제 예제가 된다.
- "Python 타입 힌트 완전 정복" 온라인 강의.
- "AI 에이전트 코드 검증 파이프라인 구축" 워크샵.
- "AGENTS.md 작성법" 블로그 시리즈 및 뉴스레터.
- 텐서 shape 타입 검증 니치 강의.
- 초기 비용이 없고, 다른 사업의 마케팅 채널로 작동한다.

### 5) PR 타입 리뷰 봇

기본 GitHub Action과의 차별점은 base 대비 신규 오류만 보여주는 델타 방식, 커버리지 이력 추적,
LLM 수정 제안, shape 회귀 감지, 조직 전체 대시보드, 머지 정책 강제다.

### 6) 타입 건강도 대시보드 (B2B)

개발자는 도구를 원하고 관리자는 지표를 원한다. 조직 전체의 커버리지 시계열, 팀별 비교,
타입 안전성 점수, 리스크 높은 모듈 랭킹, 목표 대비 진행률을 제공한다.
기술 부채 감축 예산을 정당화하는 근거 자료로 팔린다.

### 7) 웹 Python 플레이그라운드

`pyrefly_wasm` 덕분에 서버 비용 없이 운영 가능하다. 임베드형 학습 위젯,
타입 검사까지 채점하는 코딩 테스트 플랫폼, 라이브러리 문서용 인터랙티브 예제,
타입 퍼즐 게임 등으로 확장할 수 있다.

### 실행 순서 제안

1. 0~1개월: MCP 서버 오픈소스 공개 + 기술 블로그로 포지셔닝 (비용 없음)
2. 1~3개월: 마이그레이션 컨설팅으로 현금 흐름 확보, 교육 콘텐츠로 인바운드 형성
3. 3~6개월: shape 검증 SaaS MVP 개발, 컨설팅 고객을 첫 구독 고객으로 전환
4. 6~12개월: 대시보드와 PR 봇으로 제품 라인 확장, 엔터프라이즈 계약 시도

핵심 원칙은 엔진을 다시 만들지 않는 것이다. Pyrefly를 그대로 활용하고,
사용자 경험과 워크플로우, 도메인 지식으로 가치를 만든다. 차별점은 텐서 shape 검증과
AI 에이전트 통합 두 축에 있다.
