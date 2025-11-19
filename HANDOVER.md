# Spring Cloud Platform 인수인계 가이드

본 문서는 `spring-cloud-platform` 모노레포를 인수받는 담당자가 빠르게 이해하고 운영·개발 업무를 지속할 수 있도록 정리한 가이드입니다. 서비스 전반의 구성요소, 실행 절차, 필수 인프라, 보안 주의사항, 운영 체크포인트를 포함합니다.

---

## 1. 프로젝트 개요
- **목적**: Spring Cloud Netflix & Gateway 기반 MSA 데모 플랫폼. Config Server를 통한 통합 설정, Eureka 기반 서비스 디스커버리, Gateway를 통한 트래픽 라우팅 및 JWT 인증, API 라우트 관리 UI, 관측 도구 데모까지 포함합니다.
- **주요 구성요소**
  - Spring Boot 3.5 / Spring Cloud 2025 릴리즈
  - Java 21 Toolchain (Gradle Wrapper 사용)
  - 프런트엔드: Portal(Thymeleaf), Developer Portal(React + Vite, Gradle 연동)
  - 인프라: Docker Compose로 RabbitMQ, Postgres, Keycloak, Grafana/Prometheus/Tempo/Loki/Promtail, Zipkin
  - 설정 저장소: `scg_config` (내장 Git 저장소 및 원격 Git 연동)

---

## 2. 사전 준비물
- **OS**: Windows 10+, macOS, 또는 Linux (개발용은 Windows PowerShell 스크립트 기준으로 작성됨)
- **필수 설치**
  - Java 21 (Adoptium Temurin 권장) – `JAVA_HOME` 설정 필요
  - Docker Desktop (compose v2 포함) – `withDockerDependencies` 옵션 사용 시 필수
  - Node.js 20.x + npm 10.x (Developer Portal 프런트엔드 빌드용; Docker 빌드 시 자동 설치되지만 로컬 개발 시 필요)
  - Git 2.4+ (Config Repo push 기능 사용 시)
  - PowerShell 7 이상 (Windows에서 스크립트 실행)
- **권장 도구**
  - IDE: IntelliJ IDEA 또는 VS Code (Lombok 플러그인 활성화)
  - HTTP 클라이언트: Postman / HTTPie
  - 쿠버네티스 배포 예정 시 Helm, kubectl 등 추가 준비

---

## 3. 코드 구조(최상위)
| 경로 | 설명 |
| --- | --- |
| `api-admin-service/` | Gateway 라우팅 구성 Git repo 관리 및 UI 제공 (Thymeleaf) |
| `api-gateway/` | Spring Cloud Gateway. JWT 검증 필터, CORS, Resilience4J 적용 |
| `common-libs/` | 공용 DTO 모듈 (다른 서비스에서 `project(':common-libs')` 의존) |
| `config-server/` | Spring Cloud Config Server (Native + Git composite backend) |
| `discovery-server/` | Eureka Server |
| `portal-web/` | 통합 Portal (Thymeleaf) – 서비스 상태 확인 및 샘플 API 호출 |
| `developer-portal/` | React 기반 Dev Portal + Spring Security OAuth2 Client |
| `services/` | 도메인 서비스 (backend/inventory/order) |
| `infra/` | Keycloak 초기화 스크립트, Observability 설정 |
| `scg_config/` | Gateway 라우팅/서비스 설정용 Git 저장소 (로컬 clone 포함) |
| `start-platform.ps1` | 로컬 개발용 일괄 기동 스크립트 |
| `stop-platform.ps1` | 주요 포트 기반 프로세스 종료 스크립트 |
| `docker-compose.yml` | 전체 스택 컨테이너 실행 정의 |

---

## 4. 모듈별 상세
### 4.1 Config Server (`config-server`)
- 포트: 기본 `8888` (`CONFIG_SERVER_PORT` 환경변수로 변경)
- 프로필: composite (Native + Git)
  - Native: `classpath:/config` + `../scg_config`
  - Git: `https://github.com/ehdqhr3408/MSA-DEMO.git` (main 브랜치, 필요 시 자격증명 주입)
- Spring Cloud Bus (RabbitMQ) 옵션 지원. `SPRING_CLOUD_BUS_ENABLED=true` 시 actuation `/actuator/busrefresh` 사용 가능.

### 4.2 Discovery Server (`discovery-server`)
- 포트: 기본 `8761`
- Eureka Server. 모든 클라이언트 서비스는 Config Server 설정을 통해 auto-register.

### 4.3 Domain Services (`services/…`)
- **backend-service**
  - 샘플 REST API (`/api/backend/test`, `/api/backend/auth`)
  - Resilience4J CircuitBreaker + Retry 사용 (`backend-api`)
  - Micrometer Tracing(Zipkin) 및 Bus 적용
  - `backend.greeting` 설정으로 응답 메시지 제어
- **inventory-service**
  - H2 메모리 DB 기반 (`inventorydb`)
  - `/inventory/availability` 제공, 재고 정보 조회
- **order-service**
  - `/orders` 처리 (상세 로직은 Inventory 연동 예제)
  - Resilience4J CircuitBreaker (`orderServiceCircuit`)
  - JWT 리소스 서버 설정

### 4.4 Gateway (`api-gateway`)
- 포트: 기본 `8080`
- 주요 기능
  - Eureka 기반 서비스 디스커버리 라우팅 + 정적/동적 라우트 병행
  - `JwtTokenFilter` (`gateway/filter/JwtTokenFilter.java`)에서 JWKS 캐시 로드 및 검증
  - 글로벌 CORS 설정 (`config/api-gateway.yml`)
  - Resilience4J CircuitBreaker (`backendCircuitBreaker`)
  - Actuator `/actuator/gateway/**` 공개 (refresh 포함)
- 동적 라우트 소스: `scg_config/routes/**/*` (YAML) – API Admin Service가 관리

### 4.5 API Admin Service (`api-admin-service`)
- 포트: 기본 `8095`
- 역할:
  - Gateway 라우트 CRUD UI/REST 제공 (`RouteConfigFileService`)
  - 로컬 Git Repo (`ADMIN_CONFIG_REPO_PATH`, 기본 `../scg_config`) 읽기/쓰기
  - 원격 Git push 옵션 (`ADMIN_CONFIG_REPO_PUSH_ENABLED`)
  - Gateway Refresh 호출 (`GATEWAY_REFRESH_URL`)
- 종속성: RabbitMQ(Cloud Bus), Prometheus Metrics, Zipkin/OTel Export optional
- DB 없이 JGit 기반 파일 관리

### 4.6 Portal Web (`portal-web`)
- 포트: 기본 `8090`
- Thymeleaf 기반 대시보드
  - 샘플 API 호출 UI
  - 각 서비스 헬스체크/모니터링 UI
  - Grafana/Prometheus/Zipkin 등 링크 제공

### 4.7 Developer Portal (`developer-portal`)
- 포트: 기본 `8096`
- Spring Boot WAR + React SPA (Vite 빌드)
- OAuth2 Client (Keycloak) 인증 후 Grafana, Zipkin 등 링크 제공
- Gradle 빌드 시 `npm install` → `npm run build` → 정적 리소스 복사
- 로컬 프런트 개발: `cd developer-portal/src/main/frontend && npm install && npm run dev`

### 4.8 Common Libs (`common-libs`)
- 공통 DTO (예: `ApiRouteDto`, `InventoryCheckResponse`, `OrderRequest/Response`)
- 각 서비스 build.gradle에서 `implementation project(':common-libs')`

---

## 5. 설정 및 구성 관리
### 5.1 Config Repository (`scg_config`)
- 실제 운영 라우트/서비스 설정을 파일로 보관하는 Git 저장소
- 구조:
  - 루트에 서비스별 기본 설정 (예: `backend-service.yml`, `api-gateway.yml`)
  - `routes/default/*.yml`: Gateway Route 정의 + 문서화 (`*-route.yml`, `*-docs.yml`)
  - `.git/` 포함 – 로컬에서도 Git 명령 사용 가능
- API Admin Service가 이 디렉터리를 읽고 수정 후 커밋 (author 정보 환경변수로 제어)
- 원격 저장소로 push 시 `ADMIN_CONFIG_REPO_*` 환경변수 필수 설정 (현재 docker-compose에 PAT 하드코딩 → **반드시 교체/폐기 필요**)

### 5.2 Spring Cloud Config Profiles
- `config-server/src/main/resources/config/`에 기본 설정 포함
  - `application.yml`: Eureka/Tracing 공통값
  - 서비스별 YAML 파일: 포트, OAuth2 설정, Resilience4J, CORS 등
- 로컬 개발 시 기본적으로 Native 경로(`../scg_config`)가 우선 적용되므로 변경 사항은 즉시 반영됨.

### 5.3 비밀정보 관리
- 현재 `docker-compose.yml`에 GitHub Personal Access Token이 하드코딩되어 있습니다.
  - **즉시 새 토큰으로 교체하고 `.env` 또는 Docker Secret로 분리**하십시오.
  - PAT 권한은 `repo` 범위를 최소화해야 합니다.
- Keycloak 초기 비밀번호, Developer Portal client secret(`changeme`)도 운영 배포 전 변경 필요.

---

## 6. 로컬 실행 가이드
### 6.1 PowerShell 스크립트 (`start-platform.ps1`)
```powershell
# 전체 서비스 + Docker 의존성 (RabbitMQ, Postgres, Keycloak, Observability)
powershell -ExecutionPolicy Bypass -File .\start-platform.ps1 -WithDockerDependencies

# Config Server만 실행
powershell -ExecutionPolicy Bypass -File .\start-platform.ps1 -ConfigServerOnly

# Gateway 제외하고 기동
powershell -ExecutionPolicy Bypass -File .\start-platform.ps1 -NoApiGateway
```
- 스크립트 동작:
  - Gradle Wrapper(`gradlew.bat`) 확인 후 모듈 별 `bootRun` 수행
  - 각 서비스는 개별 PowerShell 창에서 실행되어 로그 확인이 용이
  - 충돌 포트 탐지 후 자동 증분 할당 (예: 8080 사용 중이면 8081로 이동)
  - `-WithDockerDependencies` 사용 시 `docker compose up -d` 호출
- 공통 환경변수 전달
  - `CONFIG_SERVER_URI`, `DISCOVERY_SERVER_PORT`, `SPRING_CLOUD_BUS_ENABLED`, `GATEWAY_SERVER_PORT` 등 서비스 간 접속 정보 주입

### 6.2 서비스 중지 (`stop-platform.ps1`)
```powershell
powershell -ExecutionPolicy Bypass -File .\stop-platform.ps1
```
- 주요 포트(8888, 8761, 8080, 8082, …)의 프로세스를 kill
- Java 프로세스 중 IDE(LSP) 제외하고 모두 정리

### 6.3 개별 서비스 실행
```bash
./gradlew :config-server:bootRun
./gradlew :api-gateway:bootRun
./gradlew :services:backend-service:bootRun
```
- Developer Portal 프런트만 개발할 경우:
  - `cd developer-portal/src/main/frontend`
  - `npm install`
  - `npm run dev` (Vite Dev Server 5173)

---

## 7. Docker 기반 실행
### 7.1 전체 스택
```bash
docker compose up -d
```
- 빌드 인수(`GRADLE_TASK`, `MODULE_DIR`)를 통해 각 모듈별 JAR/WAR 생성 후 런타임 이미지(`eclipse-temurin:21-jre-jammy`)로 기동
- 주요 서비스 및 포트:
  - `config-server` (8888)
  - `discovery-server` (8761)
  - `api-gateway` (8080)
  - `api-admin-service` (8095)
  - `portal-web` (8090)
  - `developer-portal` (8096)
  - `backend-service` (8082), `inventory-service` (8083), `order-service` (8084)
  - `keycloak` (8081), `rabbitmq` (5672/15672), `postgres` (5432), `grafana` (3000), `prometheus` (9090), `tempo` (3200/4317/4318), `loki` (3100), `zipkin` (9411)
- 볼륨
  - `postgres-data`: Postgres 영속 데이터
  - `./scg_config:/app/config/scg_config`: Config Repo를 컨테이너와 공유
  - `./runtime:/var/log/msa`: Promtail 로그 스크랩 대상

### 7.2 컨테이너 정리
```bash
docker compose down
docker compose down -v  # 볼륨 삭제 포함
```

### 7.3 Dockerfile 커스터마이징
- `Dockerfile`은 BUILD 단계에서 Node.js 20과 Yarn을 설치하여 Developer Portal 빌드를 지원
- 환경변수 `GRADLE_TASK`, `MODULE_DIR`로 대상 모듈 전환 가능
- `JAVA_OPTS` 환경변수로 런타임 JVM 옵션 주입

---

## 8. 인증 및 보안 구성
### 8.1 Keycloak (`infra/keycloak`)
- 이미지: `quay.io/keycloak/keycloak:24.0` (dev 모드)
- 엔드포인트: `http://localhost:8081`
- 관리자 계정: `admin / admin` (docker-compose 환경변수 `KEYCLOAK_ADMIN[_PASSWORD]`)
- 초깃값:
  - Realm `master`
  - Clients: `gateway`(Public), `developer-portal`(Confidential, secret `changeme`)
  - User: `test-user / 1234`
- `keycloak-entrypoint.sh`
  - 기존 H2 데이터 삭제 후 재부팅 시마다 Realm import
  - Admin CLI 로그인 재시도 로직 포함
  - Gateway/Developer Portal Client 조건부 생성/업데이트
  - 테스트 사용자 비밀번호 재설정
- **운영 시 권장**
  - Realm 분리(예: `scp`)
  - 관리자/클라이언트 비밀번호 교체
  - HTTPS Reverse Proxy 적용

### 8.2 JWT 검증
- Gateway `JwtTokenFilter`가 Keycloak JWKS를 5~10분 TTL로 캐시
- 시계 오차 허용치(`gateway.security.jwt-clock-skew` 기본 60초)
- 토큰 미검증 시 401 반환 → 외부 애플리케이션은 반드시 Authorization 헤더 제공

---

## 9. 관측 및 로깅
- **Prometheus**: `http://localhost:9090` – `infra/observability/prometheus.yml`에서 스크랩 타깃 정의 (`host.docker.internal` 기준, 필요 시 수정)
- **Grafana**: `http://localhost:3000` (admin/admin). `infra/observability/grafana`에 초기 DataSource/대시보드(`msa-dashboard.json`) 자동 로드
- **Tempo / Loki / Promtail**: 분산 트레이싱 및 로그 수집. 서비스 로그를 `./runtime`에 출력하여 Promtail이 수집하도록 구조화 필요
- **Zipkin**: `http://localhost:9411` – Micrometer Tracing→Zipkin Reporter 구성
- **Management Endpoints**
  - 대부분 서비스에서 `management.endpoints.web.exposure.include`를 통해 `health`, `info`, `metrics`, `prometheus`, `refresh` 등을 노출
  - Portal Web의 `/monitor/*` API가 각 health/metrics endpoint를 주기적으로 조회해 대시보드 제공

---

## 10. 개발 워크플로우
1. **로컬 개발 시작**
   - `./gradlew clean build`로 전체 검증
   - `./start-platform.ps1 -WithDockerDependencies`로 필요한 인프라+서비스 기동
2. **라우트 변경**
   - Portal Admin UI (`http://localhost:8095`) 또는 REST API로 라우트 생성/수정
   - 변경 시 `scg_config/routes/...`에 YAML 생성 → Git 커밋 발생
   - Gateway Refresh 버튼 또는 `/actuator/gateway/refresh` 호출로 즉시 반영
3. **설정 변경**
   - `scg_config/*.yml` 수정 후 Config Server Refresh (`/actuator/refresh` 또는 버스 사용 시 `/actuator/busrefresh`)
4. **프런트 개발**
   - Developer Portal: `npm run dev`, 변경 시 Gradle `bootWar` 빌드 필요
   - Portal Web: Thymeleaf 템플릿 수정 후 `bootRun` 재시작
5. **테스트**
   - `./gradlew test` (하위모듈 포함)
   - 특정 모듈: `./gradlew :services:backend-service:test`
6. **로그 확인**
   - PowerShell 실행 창 또는 `runtime` 폴더 (Promtail 수집 대상)
   - Gateway 필터 로그: `api-gateway` 모듈 콘솔

---

## 11. 배포/릴리스 전략
- **Jar/WAR 아티팩트**
  - `./gradlew :module:bootJar` 또는 `bootWar` (Developer Portal)
  - 빌드 산출물: `<module>/build/libs/*.jar|war`
- **컨테이너 이미지**
  - `docker build --build-arg GRADLE_TASK=:api-gateway:bootJar --build-arg MODULE_DIR=api-gateway -t scp-api-gateway .`
  - 다중 모듈 이미지를 따로 빌드하려면 ARGS 변경
- **CI/CD 제안**
  - 멀티-모듈로 인해 Gradle 캐시를 공유하는 CI 환경 권장(GitHub Actions, Jenkins 등)
  - Config Repo(`scg_config`)는 별도 리포지토리로 분리하여 관리하는 것을 권장

---

## 12. 운영 체크리스트 & 주의사항
- [ ] **PAT/비밀번호 교체**: docker-compose에 포함된 GitHub PAT, Keycloak secrets, Developer Portal secret을 즉시 교체
- [ ] **Config Repo 백업**: `scg_config` 디렉터리를 주기적으로 백업하거나 원격 저장소 미러링
- [ ] **RabbitMQ/Zipkin 등 의존성**: Bus 기능 사용 시 RabbitMQ 기동 여부 확인. Zipkin/Tempo 접근 가능성 점검
- [ ] **포트 충돌**: 로컬 실행 시 이미 사용 중인 포트가 있으면 Start 스크립트가 대체 포트를 지정 → 콘솔 출력 확인 필수
- [ ] **프런트 빌드 산출물**: Developer Portal는 Gradle 빌드 전 `npm install` 캐시 확인 (CI에서는 Node 20 설치 필요)
- [ ] **보안 모드 전환**: 운영 환경에서는 HTTPS, 프로덕션 Keycloak 모드, 강력한 CORS/CSRF 전략 적용 필요
- [ ] **로그 경로**: Promtail은 `./runtime` 경로를 읽습니다. 서비스에서 필요한 로그를 해당 경로에 저장하도록 설정 검토
- [ ] **Grafana 인증**: 기본 admin/admin 계정 사용 중단, 조직/대시보드 권한 정의
- [ ] **Helth/Actuator 노출**: 외부 노출 시 네트워크 레벨에서 보호 (예: API Gateway 뒤에 배치)

---

## 13. 참고 링크 & 문서
- Spring Cloud Gateway Docs: https://spring.io/projects/spring-cloud-gateway
- Spring Cloud Config Docs: https://spring.io/projects/spring-cloud-config
- Resilience4J: https://resilience4j.readme.io/
- Keycloak Admin CLI: https://www.keycloak.org/docs/latest/server_admin/#admin-cli
- Grafana Provisioning: https://grafana.com/docs/grafana/latest/administration/provisioning/

추가 문의 사항이나 인수인계 미비점은 기존 개발 담당자 또는 DevOps 담당자와 협의해 갱신하시기 바랍니다.

