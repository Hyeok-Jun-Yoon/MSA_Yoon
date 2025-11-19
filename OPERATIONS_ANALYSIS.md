# Spring Cloud Platform 운영/배포 분석

운영 환경 배포를 위한 `spring-cloud-platform` 리포지토리 구성과 주요 파일의 역할, 상업용 배포 시 고려해야 할 항목을 정리한 문서입니다. (2025-11 기준)

---

## 1. 아키텍처 개요
- **구성 방식**: Spring Boot 멀티모듈(Gradle) + Spring Cloud Netflix & Gateway.
- **핵심 요소**
  - Config Server (`config-server`)와 Git 기반 설정 저장소(`scg_config/`).
  - Eureka Discovery (`discovery-server`), Spring Cloud Gateway (`api-gateway`).
  - API Admin Service (`api-admin-service`)로 Gateway 라우트 YAML 관리.
  - 도메인 백엔드(backend/inventory/order) + Web Portal(Thymeleaf) + Developer Portal(React+Spring).
  - Observability 스택 (Grafana/Prometheus/Tempo/Loki/Promtail/Zipkin) + Keycloak + RabbitMQ + Postgres (모두 `docker-compose.yml`).
- **빌드 도구**: Gradle Wrapper (Java 21 toolchain) / Node.js 20 (Developer Portal).

---

## 2. 모듈별 역할과 파일
| 모듈 | 주요 소스 | 역할/설명 |
| --- | --- | --- |
| Config Server | `config-server/src/main/java/.../ConfigServerApplication.java`, `config/application.yml` | Composite backend (classpath + `../scg_config`). RabbitMQ Bus 옵션. |
| Discovery Server | `discovery-server/.../DiscoveryServerApplication.java` | Eureka 서버 (포트 8761). |
| API Gateway | `api-gateway/src/main/java/com/example/platform/gateway` | `JwtTokenFilter.java`(JWT 검증), `PostLoggingFilter.java`(응답로그), `RouteLocator` 대신 파일 기반 라우팅. |
| API Admin Service | `api-admin-service/src/main/java/com/example/apiadmin` | `RouteConfigFileService.java` (YAML CRUD + Git commit/push), `RouteViewController`(Thymeleaf). |
| Portal Web | `portal-web/...` | 모니터링 뷰, API 호출 UI, Gateway/Service health 확인. |
| Developer Portal | `developer-portal/src/main/java` + `src/main/frontend` | React 빌드 결과를 Spring Boot WAR에 포함. OAuth2 클라이언트로 Grafana/Zipkin 링크 제공. |
| Domain Services | `services/backend-service`, `inventory-service`, `order-service` | 예제 REST API, Resilience4J 설정, JWT ResourceServer, Inventory H2 DB 등. |
| Common-libs | `common-libs/src/main/java/com/example/common/dto` | 공유 DTO (ApiRouteDto 등). 모든 서비스가 `implementation project(':common-libs')`로 참조. |
| Infra Assets | `infra/keycloak/*`, `infra/observability/*` | Keycloak realm export, Grafana/Tempo/Loki/Promtail/Prometheus 설정. |
| Scripts | `start-platform.ps1`, `stop-platform.ps1` | PowerShell 기반 로컬 기동/중지 자동화. |

---

## 3. Config Repo (`scg_config/`)
- **구성**: 서비스별 YAML + `routes/<group>/*.yml`.
- **관리 방식**: API Admin Service가 로컬 경로(`ADMIN_CONFIG_REPO_PATH`, 기본 `../scg_config`)를 직접 수정 후 Git commit/push.
- **Remote 연동**: `ADMIN_CONFIG_REPO_REMOTE_URI`, `USERNAME`, `PASSWORD`가 `docker-compose.yml`에 하드코딩 → 상업용 배포 시 Secret 처리 필수.
- **Gateway 연동**: Gateway는 `GATEWAY_ROUTES_PATH` (`/app/config/scg_config/routes`)를 마운트해 라우트 로드 후 `/actuator/gateway/refresh`로 즉시 반영.

---

## 4. 빌드 & 테스트 파이프라인
1. **Gradle 멀티모듈 설정**: `settings.gradle`에 모든 모듈 명시.
2. **루트 빌드 설정**: `build.gradle`에서 Spring Boot/Cloud BOM, Java 21 toolchain 적용.
3. **CI 추천 순서**
   ```bash
   ./gradlew clean build         # 모든 모듈 (커버리지 필요 시 test 포함)
   ./gradlew :developer-portal:npmInstall  # node_modules 캐시
   ./gradlew :developer-portal:bootWar     # React 빌드 포함
   ```
4. **도커 빌드**: `Dockerfile`은 ARG `GRADLE_TASK` + `MODULE_DIR` 조합으로 공통 빌더 이미지를 재사용.
5. **start-platform.ps1**: 로컬 개발 시 각 모듈을 개별 PowerShell 창에서 `gradlew :module:bootRun`으로 구동하고, 옵션으로 Docker 의존 서비스(`-WithDockerDependencies`) 기동.

---

## 5. Docker Compose 스택 (`docker-compose.yml`)
- **서비스**: RabbitMQ, Postgres, Zipkin, Keycloak, Tempo, Loki, Promtail, Prometheus, Grafana, Config/Discovery/Gateway/API-Admin/Portal-Web/Developer-Portal/Backend/Inventory/Order.
- **환경 변수**: `x-config-client-env` 앵커로 대부분 서비스에 Config/Eureka/RabbitMQ 등 공통값 주입.
- **볼륨**:
  - `postgres-data` (데이터 지속화)
  - `./scg_config:/app/config/scg_config` (Config repo 공유)
  - `./runtime:/var/log/msa` (Promtail 로그 수집)
- **보안 이슈**: GitHub PAT, DB/Keycloak 계정이 하드코딩. 상업용 배포 전 `.env`/Secret Store로 이동 필수.

---

## 6. 보안 및 인증
- **Keycloak** (`infra/keycloak`): Realm export + custom entrypoint (`keycloak-entrypoint.sh`)로 client/user 초기 구성. dev credentials (admin/admin, client secret `changeme`) → 운영 전 교체.
- **JWT 검증**: `JwtTokenFilter`가 Keycloak JWKS 캐시(TTL `gateway.security.jwks-cache-ttl`, 기본 5분) 사용. 시계 오차 허용(`jwt-clock-skew` 기본 60초).
- **API Admin push**: PAT가 코드에 드러나 있으므로 최소 권한 토큰 발급 및 Vault/Secret Manager 연계 필요.
- **HTTPS**: 현재 모든 서비스 HTTP. 운영 환경에서는 Gateway/Keycloak 앞단에 TLS Termination Proxy 또는 Spring Boot HTTPS 설정 필요.

---

## 7. Observability
- **Prometheus** (`infra/observability/prometheus.yml`): Config/Discovery/Gateway/Services/Portal 등을 `host.docker.internal` 기준으로 스크랩.
- **Grafana** (`infra/observability/grafana`): 데이터소스/대시보드 자동 프로비저닝. 기본 admin/admin + Anonymous Viewer 활성화 → 운영 시 인증 강화 필요.
- **Tempo/Loki/Promtail**: 분산 추적/로그 파이프라인. Promtail은 `./runtime` 로그 디렉터리를 읽어 Loki로 전달.
- **Zipkin**: `http://localhost:9411` 에 Micrometer Tracing 결과 전송.
- **Portal Web**: `/monitor/*` 엔드포인트에서 health/metrics 확인 및 Grafana/Zipkin 링크 제공.

---

## 8. 배포 고려사항 (상업용)
1. **비밀관리**
   - PAT/비밀번호/Client Secret을 `.env` 또는 Secret Manager로 이동.
   - PowerShell 스크립트와 Docker Compose는 `.env`를 참조하도록 수정.
2. **네트워크/TLS**
   - Gateway/Keycloak/Observability에 HTTPS 도입 (Ingress Controller 또는 Reverse Proxy).
   - Grafana/Prometheus는 사설망에서만 접근하도록 방화벽/보안 그룹 설정.
3. **확장성**
   - Config/Eureka/Gateway/API/Backend 등을 컨테이너 오케스트레이션(Kubernetes 등)으로 배포할 경우 readiness/liveness probe 정의 필요.
   - `Dockerfile` 기반 이미지를 CI에서 모듈별로 빌드 → Registry에 push → Helm/Manifest 구성.
4. **데이터**
   - Postgres 초기 스크립트(`api-admin-service/src/main/resources/db/migration/V1__init.sql`) 검토 후 운영 데이터 스키마 설계.
   - Inventory 서비스의 H2 대신 외부 DB로 교체 고려.
5. **로그/모니터링**
   - Promtail 수집 경로(`./runtime`)를 운영 서버 경로에 맞춰 조정.
   - Grafana Alerting/Tempo retention 등 운영 파라미터 튜닝.
6. **테스트**
   - 모듈별 통합 테스트 부재 → API Gateway 필터, RouteConfigFileService 등 핵심 로직에 단위/통합 테스트 추가 권장.

---

## 9. 파일별 참고 포인트
- `settings.gradle`: 모든 모듈 선언 및 네이밍.
- `build.gradle`: 공통 dependency management, Java toolchain 설정.
- `api-gateway/src/main/java/.../JwtTokenFilter.java`: 상업용 JWT 검증 로직 핵심 (JWKS 캐시/검증/로그).
- `api-admin-service/src/main/java/.../RouteConfigFileService.java`: 라우트 CRUD + Git 연동 로직.
- `start-platform.ps1` / `stop-platform.ps1`: 로컬 기동/중지 자동화 및 포트 충돌 처리.
- `Dockerfile`: 모듈별 빌더/런타임 이미지를 하나로 통합하는 전략 (CI에서 재사용).
- `docker-compose.yml`: 전체 인프라/서비스 의존성, 비밀값 확인.
- `.vscode/settings.json`: Lombok agent 설정 (IDE 오류 방지).
- `HANDOVER_v2.md`: 인수인계용 운영 절차 요약 (본 문서와 중복되는 항목 참고).

---

## 10. 배포 체크리스트
- [ ] Secrets → `.env` / Secret Manager / Vault.
- [ ] Keycloak realm & Grafana admin 비밀번호 변경.
- [ ] HTTPS + 네트워크 보안 구성.
- [ ] Config Repo Remote 인증정보 재구성 (사내 Git 등).
- [ ] Observability 접근 제어 및 알람 시나리오 수립.
- [ ] Docker 이미지 빌드/Registry push 자동화 (CI).
- [ ] Kubernetes/VM 배포 전략 결정 후 Helm/Ansible 등 템플릿 작성.
- [ ] 운영 로그/백업 정책 정의 (`scg_config`, Postgres, runtime logs).
- [ ] 테스트 시나리오/QA 체크 (JWT 필터, 라우트 CRUD, Admin UI, Portal UI, Developer Portal OAuth 등).

이 문서를 기반으로 각 모듈 담당자와 보안/Infra 팀과 협력해 상업용 배포를 위한 세부 절차 및 자동화 스크립트를 확정하시기 바랍니다.
