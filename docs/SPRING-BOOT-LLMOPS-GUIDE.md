# Spring Boot 기반 LLMOps 플랫폼 개발 가이드 (AI 전달용)

## 1) 목표/범위
이 문서는 **Dify 핵심 기능(플러그인 제외)**을 벤치마킹하여, `Spring Boot + Spring AI + MariaDB + Keycloak(OIDC SSO)` 스택으로 구현하기 위한 실행 가능한 청사진입니다.

- 필수: 워크플로우, 프롬프트 관리, RAG, 앱 배포/실행, 로그/관측, 모델 관리, 멀티테넌시(워크스페이스), API Key
- 제외 가능: Dify 플러그인 마켓/플러그인 런타임
- 보안: **RBAC 없이 단순 인증(로그인 사용자만 접근)**

---

## 2) 권장 아키텍처

```text
[Browser: HTML+Vanilla JS]
  -> [Spring Boot MVC + REST]
     -> [Security: Spring Security OAuth2 Client/OIDC + Keycloak]
     -> [Domain Modules]
        - Console(App/Prompt/Workflow/RAG 관리)
        - Runtime(Chat/Workflow 실행)
        - Knowledge(문서 업로드/청킹/임베딩/검색)
        - Model Provider(OpenAI/Azure/Bedrock/Ollama...)
        - Observability(실행로그/토큰/지연)
     -> [MariaDB]
     -> [Object Storage(S3/MinIO)] (원문파일)
     -> [Vector Store(pgvector/Elastic/OpenSearch/Qdrant 중 택1)]
```

> 참고: 벡터 저장소는 MariaDB 자체로는 대규모 유사도 검색이 제한적이므로, RAG 정확도/성능을 위해 별도 벡터 스토어를 권장합니다.

---

## 3) 모듈 구조 (Monolith 우선)

```text
src/main/java/com/kaia/llmops
  ├─ common
  │   ├─ config
  │   ├─ error
  │   └─ util
  ├─ auth                  # Keycloak OIDC 로그인
  ├─ workspace             # 테넌트/프로젝트 경계
  ├─ model                 # LLM/Embedding/Provider 설정
  ├─ app                   # Dify의 App 개념 (Chat/Agent/Workflow 앱)
  ├─ prompt                # Prompt 템플릿/버전/파라미터
  ├─ workflow              # 노드/엣지 기반 워크플로우 엔진
  ├─ knowledge             # 문서, 청킹, 임베딩, 인덱싱
  ├─ retrieval             # 하이브리드 검색(BM25+Vector)
  ├─ runtime               # 대화 실행, 스트리밍, tool call
  ├─ observability         # 실행로그, 비용, 토큰사용량
  └─ api                   # 공개 API key 실행 엔드포인트
```

---

## 4) 핵심 도메인 모델 (Dify 대응)

- `Workspace` : 팀/프로젝트 단위
- `User` : Keycloak subject(`sub`) 매핑 사용자
- `App` : Chatbot / Workflow / Completion 앱
- `PromptTemplate` : 시스템/사용자 프롬프트, 변수, 버전
- `Workflow`, `WorkflowNode`, `WorkflowEdge` : 그래프 실행 모델
- `Dataset`, `Document`, `DocumentChunk`, `Embedding` : RAG 파이프라인
- `Conversation`, `Message`, `Run` : 실행 이력
- `ModelProvider`, `ModelConfig` : 모델 공급자/모델별 설정
- `ApiKey` : 서버 간 호출 키
- `TraceLog` : 지연, 토큰, 에러, 비용

---

## 5) Keycloak OIDC SSO 설계 (Authorization Code Flow)

### 5.1 Keycloak 설정
1. Realm 생성: `kaia`
2. Client 생성: `llmops-console`
   - Access Type: `confidential`
   - Valid Redirect URI: `http://localhost:8080/login/oauth2/code/keycloak`
   - Web Origins: `http://localhost:8080`
3. 사용자 생성 및 로그인 테스트

### 5.2 Spring Security 설정 포인트
- `oauth2Login()` 활성화
- 인증 필요 경로: `/console/**`, `/api/**`
- 공개 경로: `/`, `/assets/**`, `/login/**`
- JWT 세션: 브라우저 로그인은 세션 기반, API는 필요시 Bearer(JWT) 허용
- 권한 체크는 생략하고 `authenticated()`만 사용

### 5.3 application.properties 예시
```properties
spring.security.oauth2.client.registration.keycloak.client-id=llmops-console
spring.security.oauth2.client.registration.keycloak.client-secret=${KEYCLOAK_CLIENT_SECRET}
spring.security.oauth2.client.registration.keycloak.scope=openid,profile,email
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=authorization_code

spring.security.oauth2.client.provider.keycloak.issuer-uri=http://localhost:8081/realms/kaia

spring.datasource.url=jdbc:mariadb://localhost:3306/llmops
spring.datasource.username=llmops
spring.datasource.password=llmops
spring.jpa.hibernate.ddl-auto=update
spring.jpa.open-in-view=false
```

---

## 6) HTML + JavaScript(무프레임워크) UI 구성

- `/` : 랜딩 + 로그인 버튼
- `/console/apps` : 앱 목록/생성
- `/console/prompts` : 템플릿 목록/버전 관리
- `/console/workflows/{id}` : 노드 그래프 편집(초기에는 JSON 편집기)
- `/console/knowledge` : 파일 업로드/인덱싱 상태
- `/console/playground` : 모델 테스트 + 스트리밍 응답
- `/console/observability` : 실행 로그 대시보드

> 그래프 UI는 초기에 Canvas/SVG 기반 단순 구현 후, 추후 라이브러리 도입 가능.

---

## 7) Dify 핵심 기능별 구현 전략

## 7.1 App Builder
- App 유형: `CHAT`, `WORKFLOW`, `COMPLETION`
- 시스템 프롬프트 + 파라미터(top_p, temperature, max_tokens)
- 공개 API endpoint 발급(`/v1/apps/{appId}/chat`)

## 7.2 Prompt 관리
- 템플릿 변수(`{{user_input}}`, `{{context}}`) 파서
- 버전 관리: `v1, v2 ...` + 활성 버전 포인터
- Prompt 검증: 필수 변수/길이/금칙어 룰

## 7.3 Workflow 엔진
- 노드 타입(초기 권장)
  - Start
  - LLM
  - Knowledge Retrieval
  - Condition/Branch
  - Template
  - HTTP Request
  - End
- 실행 방식: DAG topological order + 분기 merge
- 노드별 입출력 JSON Schema 검증
- 재시도/타임아웃/중단 토큰 지원

## 7.4 RAG 파이프라인
1. 문서 업로드(S3/MinIO)
2. 파서(PDF/MD/TXT)
3. 청킹(문단/토큰 기반)
4. 임베딩 생성(Spring AI EmbeddingModel)
5. 벡터 저장
6. 검색(Vector + BM25 하이브리드)
7. Re-ranking(선택)
8. Prompt에 context 결합

## 7.5 Runtime(대화 실행)
- SSE 스트리밍(`/v1/chat/stream`)
- 대화 메모리: 최근 N턴 + 요약 메모리
- Tool calling: HTTP tool adapter
- 안전장치: 토큰 상한/요청당 timeout

## 7.6 Observability
- 실행 단위 `trace_id`
- 저장 항목: latency, prompt tokens, completion tokens, model, 에러코드
- 검색 필터: 앱, 기간, 에러 여부

---

## 8) DB 스키마 초안 (MariaDB)

```sql
create table users (
  id bigint primary key auto_increment,
  keycloak_sub varchar(128) not null unique,
  email varchar(255),
  name varchar(120),
  created_at datetime not null default current_timestamp
);

create table workspaces (
  id bigint primary key auto_increment,
  name varchar(120) not null,
  created_by bigint not null,
  created_at datetime not null default current_timestamp
);

create table apps (
  id bigint primary key auto_increment,
  workspace_id bigint not null,
  name varchar(120) not null,
  app_type varchar(32) not null,
  active boolean not null default true,
  created_at datetime not null default current_timestamp
);

create table prompt_templates (
  id bigint primary key auto_increment,
  app_id bigint not null,
  version int not null,
  system_prompt text,
  user_prompt text,
  variables_json json,
  is_active boolean not null default false,
  created_at datetime not null default current_timestamp
);

create table datasets (
  id bigint primary key auto_increment,
  workspace_id bigint not null,
  name varchar(120) not null,
  embedding_model varchar(120) not null,
  created_at datetime not null default current_timestamp
);

create table documents (
  id bigint primary key auto_increment,
  dataset_id bigint not null,
  filename varchar(255) not null,
  status varchar(32) not null,
  object_key varchar(255),
  created_at datetime not null default current_timestamp
);

create table document_chunks (
  id bigint primary key auto_increment,
  document_id bigint not null,
  chunk_index int not null,
  content text not null,
  token_count int,
  metadata_json json
);

create table conversations (
  id bigint primary key auto_increment,
  app_id bigint not null,
  user_id bigint,
  started_at datetime not null default current_timestamp
);

create table messages (
  id bigint primary key auto_increment,
  conversation_id bigint not null,
  role varchar(32) not null,
  content longtext not null,
  token_count int,
  created_at datetime not null default current_timestamp
);

create table run_logs (
  id bigint primary key auto_increment,
  trace_id varchar(64) not null,
  app_id bigint,
  model varchar(120),
  latency_ms int,
  prompt_tokens int,
  completion_tokens int,
  status varchar(32),
  error_message text,
  created_at datetime not null default current_timestamp,
  index idx_trace_id(trace_id),
  index idx_created_at(created_at)
);
```

---

## 9) Spring Boot 구현 체크리스트

### 9.1 프로젝트 생성
- Java 17+, Spring Boot 3.x
- 의존성
  - Spring Web, Validation, Security
  - OAuth2 Client, OAuth2 Resource Server
  - Spring Data JPA, MariaDB Driver
  - Spring AI (OpenAI/Bedrock/Ollama 사용에 맞춰 starter 선택)
  - Actuator

### 9.2 SecurityConfig 예시
```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .csrf(csrf -> csrf.ignoringRequestMatchers("/v1/**"))
      .authorizeHttpRequests(auth -> auth
          .requestMatchers("/", "/assets/**", "/login/**").permitAll()
          .anyRequest().authenticated())
      .oauth2Login(Customizer.withDefaults())
      .oauth2Client(Customizer.withDefaults())
      .logout(logout -> logout.logoutSuccessUrl("/"));
    return http.build();
}
```

### 9.3 OIDC 사용자 동기화
- 로그인 성공 시 `OidcUser#getSubject()`를 `users.keycloak_sub`와 upsert
- 최초 로그인 사용자 자동 등록

### 9.4 API 설계 (요약)
- `POST /api/apps`
- `GET /api/apps`
- `POST /api/prompts/{appId}/versions`
- `POST /api/workflows/{appId}/publish`
- `POST /api/datasets/{id}/documents`
- `POST /v1/chat`
- `GET /api/logs`

---

## 10) 단계별 로드맵 (권장)

### Phase 1 (2~3주)
- Keycloak SSO 로그인
- App/Prompt 기본 CRUD
- 단일 모델 호출 Playground

### Phase 2 (3~4주)
- Dataset/Document 업로드
- 청킹/임베딩/검색(RAG)
- Chat API + SSE 스트리밍

### Phase 3 (4~6주)
- Workflow 그래프 실행 엔진
- 조건 분기/HTTP 노드/템플릿 노드
- 실행 로그/비용 대시보드

### Phase 4 (2~3주)
- API Key/외부 호출 안정화
- 운영 기능(백업, 알림, 장애 대응)
- 성능 튜닝(캐시, 비동기, 배치)

---

## 11) 운영/배포 가이드

- Docker Compose 구성
  - app (Spring Boot)
  - mariadb
  - keycloak
  - minio
  - vector store(qdrant/opensearch 등)
- 필수 환경변수
  - `KEYCLOAK_CLIENT_SECRET`
  - `SPRING_DATASOURCE_URL`
  - `OPENAI_API_KEY` (또는 공급자 키)
- 모니터링
  - `/actuator/health`, `/actuator/metrics`
  - 로그: trace_id 기반 구조화(JSON)

---

## 12) 최소 완료 기준(DoD)

- [ ] Keycloak 로그인/로그아웃이 브라우저에서 정상 동작
- [ ] 인증 사용자만 콘솔/API 접근 가능
- [ ] App 생성 후 Prompt 버전 관리 가능
- [ ] 문서 업로드 후 RAG 검색 결과를 프롬프트에 주입 가능
- [ ] Chat/Workflow 실행 결과를 SSE로 수신 가능
- [ ] 실행 로그(토큰/지연/오류) 조회 가능

---

## 13) 현실적인 범위 관리 메모

Dify와 “완전히 동일”한 구현은 대규모 작업입니다. 따라서 위 구조로 시작하면,
- 기능 동등성(핵심 기능) 확보
- 기술 스택 제약(Spring Boot/Spring AI/HTML+JS/OIDC) 충족
- 이후 플러그인 생태계를 제외한 실사용 수준 플랫폼으로 확장
이 가능합니다.

