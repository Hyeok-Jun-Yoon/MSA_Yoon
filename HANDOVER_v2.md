# Spring Cloud Platform 인수인계 (v2)

`spring-cloud-platform` 모노레포의 현재 구성을 기준으로 운영/개발 담당자가 알아야 할 정보를 정리했습니다. 2025-11 기준 Gradle 빌드가 통과된 상태이며, Docker/PowerShell 스크립트도 최신 코드와 일치합니다.

---

## 1. 리포지토리 & 빌드 스택
- **언어/버전**: Java 21 (Gradle Toolchain), Node.js 20.x (Developer Portal)
- **프레임워크**: Spring Boot 3.5.6, Spring Cloud 2025.0.0, Spring Cloud Gateway, Eureka, Resilience4J
- **구성**: 멀티모듈 Gradle (`settings.gradle` 참고) + `scg_config` Git 저장소 연동
- **주요 외부 컴포넌트**: Keycloak 24, RabbitMQ 3.13, Postgres 15, Grafana/Prometheus/Tempo/Loki/Zipkin

---

## 2. 모듈 개요
| 모듈 | 경로 | 설명 |
| --- | --- | --- |
| Config Server | `config-server` | composite(native + Git) backend. 기본 8888 |
| Discovery Server | `discovery-server` | Eureka 서버. 기본 8761 |
| API Gateway | `api-gateway` | Spring Cloud Gateway + `JwtTokenFilter`(JWKS 캐시) |
| API Admin Service | `api-admin-service` | `scg_config/routes` CRUD & Gateway refresh |
| Portal Web | `portal-web` | Thymeleaf 기반 관제/대시보드 |
| Developer Portal | `developer-portal` | React(Vite) + Spring Boot WAR, OAuth2 Keycloak 클라이언트 |
| Domain Services | `services/*` | backend / inventory / order 서비스 (각각 8082~8084) |
| Common DTO | `common-libs` | 공통 DTO (예: `ApiRouteDto`) |
| Infra Assets | `infra/*` | Keycloak Realm, 관측(Tempo/Loki/Promtail/Prometheus/Grafana) 구성 |
| Config Repo | `scg_config` | Gateway 라우트 및 공통 설정 Git 저장소 (로컬 clone) |

---

## 3. 준비물 & IDE 셋업
1. **필수 도구**
   - JDK 21 (`JAVA_HOME` 필수)
   - Docker Desktop (Compose v2)
   - Node.js 20.x / npm 10.x (Developer Portal 빌드)
   - Git 2.4+
   - PowerShell 7 이상 (start/stop 스크립트)
2. **VS Code**
   - Lombok javaagent 등록: `.vscode/settings.json`의 `java.jdt.ls.vmargs` / `netbeans.jdkOptions` 이미 설정됨
   - Lombok 지원 옵션(`java.jdt.ls.lombokSupport.enabled: true`) 유지
3. **환경변수**
   - 로컬 실행 시 Config/Eureka URL, Keycloak issuer 등은 `start-platform.ps1`에서 자동 주입
   - 개별 실행 시 `CONFIG_SERVER_URI=http://localhost:8888` 등을 수동 지정

---

## 4. Config Repository (`scg_config`)
- 구조
  - 루트: 서비스별 `application-*.yml`
  - `routes/<group>/*.yml`: Gateway route 정의
  - `.git` 포함 (API Admin Service가 직접 커밋/푸시)
- Docker Compose에는 아래 값이 **하드코딩** 되어 있음
  - `ADMIN_CONFIG_REPO_REMOTE_URI`, `USERNAME`, `PASSWORD` (`ghp_tB5D...` PAT)
  - **운영 투입 전 반드시 `.env`/Secret 으로 분리하고 PAT 교체 필요**
- Gateway 라우트 CRUD
  - `RouteConfigFileService`가 YAML 파일을 읽어 DTO 변환/검증
  - 라우트 저장 시 Git commit 및 (옵션) remote push

---

## 5. 로컬 동작 절차
### 5.1 전체 기동 (PowerShell)
```powershell
powershell -ExecutionPolicy Bypass -File .\start-platform.ps1 -WithDockerDependencies
```
- Docker 스택(Observability, Keycloak, Postgres, RabbitMQ) + 모든 Spring 부트 애플리케이션 기동
- 각 서비스는 개별 PowerShell 창에서 `gradlew :module:bootRun`
- `-No<ApiName>` 스위치로 특정 모듈 제외 가능, `-ConfigServerOnly` 지원

### 5.2 종료
```powershell
powershell -ExecutionPolicy Bypass -File .\stop-platform.ps1
```
- 주요 포트(8888, 8761, 8080, 8082~8084, 8090, 8095, 8096 등) 사용 중 프로세스 종료

### 5.3 개별 실행
```bash
./gradlew :config-server:bootRun
./gradlew :api-gateway:bootRun
./gradlew :services:backend-service:bootRun
```
- Developer Portal 프런트엔드 실시간 개발:
  - `cd developer-portal/src/main/frontend`
  - `npm install && npm run dev`

---

## 6. Docker Compose 운영
1. **기동**
   ```bash
   docker compose up -d
   ```
   - `Dockerfile`은 `GRADLE_TASK` / `MODULE_DIR` ARG로 모듈별 bootJar/bootWar 빌드 후 Temurin 21 JRE 이미지에 배포
   - `./scg_config` 볼륨이 Admin Service·Gateway 컨테이너에 마운트
   - 관측 스택(Grafana/Prometheus/Tempo/Loki/Promtail/Zipkin) 포함
2. **중지**
   ```bash
   docker compose down       # 컨테이너만
   docker compose down -v    # 볼륨까지 삭제 (Postgres 초기화)
   ```
3. **환경 변경 시**
   - PAT·암호를 `.env`에 두고 `docker compose --env-file` 옵션 권장
   - Grafana admin 비밀번호, Keycloak user/realm 비밀번호 즉시 교체

---

## 7. 관측/보안 체크포인트
- **Prometheus** `http://localhost:9090`, Scrape 설정은 `infra/observability/prometheus.yml`
- **Grafana** `http://localhost:3000` (기본 admin/admin). `infra/observability/grafana`에 DataSource/Dashboard 프로비저닝
- **Tempo/Loki/Promtail**: `./runtime` 로그 디렉터리를 공유 → 운영 시 경로 조정 필요
- **Zipkin** `http://localhost:9411`, Micrometer Tracing이 Zipkin exporter 사용
- **Keycloak**
  - `infra/keycloak/keycloak-entrypoint.sh`가 realm import 및 client/user 생성
  - dev 기본 값(아이디/비밀번호, client secret `changeme`) → 운영 전 교체
- **API Gateway JWT**
  - `gateway/security` 관련 속성은 `application.yml` 및 Config Server에서 관리
  - `gateway.security.jwks-uri`, `jwks-cache-ttl`, `jwt-clock-skew` 프로퍼티 확인

---

## 8. 알려진 리스크 & 조치 필요 항목
1. **PAT/비밀번호 하드코딩** (`docker-compose.yml`)
   - GitHub PAT, Postgres 비밀번호(3408), Keycloak admin/admin 그대로 노출 → `.env` 분리 및 Secret Store 사용 권장
2. **HTTPS 미구성**
   - Gateway/Keycloak 모두 HTTP. 운영 환경에서 Reverse Proxy + TLS 필수
3. **Observability 인증 미적용**
   - Grafana anonymous access ON, Prometheus/Tempo/Loki 미보호 → 운영 전 인증 계층 필요
4. **Config Repo 원격 push**
   - Admin Service가 docker-compose 내 하드코딩 계정으로 push 수행 → 최소 권한 토큰 및 감사 로그 확보 필요
5. **Lombok 의존 IDE 설정**
   - VS Code Lombok agent 설정이 깨지면 빌드에는 영향 없지만 IDE 오류 다수 → `.vscode/settings.json` 유지/공유

---

## 9. 트러블슈팅 힌트
- **Gradle 빌드 실패 (루트 실행)**: 루트(`vscode_codex/`)가 아닌 `spring-cloud-platform/`에서 `gradlew` 실행
- **Gateway 401**: Keycloak 8081 realm/clients가 미기동 상태일 가능성. `docker compose logs keycloak` 확인 후 `JwtTokenFilter` 로그 확인
- **Route 미반영**: `api-admin-service` UI 후 `Gateway Refresh` 실행 여부 확인 (`/actuator/gateway/refresh`)
- **Developer Portal 프런트 빌드 오류**: Node 20.x 미설치 또는 `npm install` 누락. 프런트 경로에서 별도 설치 필요
- **로그 수집 미동작**: `./runtime` 퍼미션과 Promtail 타겟 경로 확인

---

## 10. 다음 담당자 To-Do
- [ ] `.env` 도입 및 모든 비밀번호/PAT/Secret 치환
- [ ] Keycloak realm을 운영 값으로 재구성 (user, client secret, TLS)
- [ ] Grafana/Prometheus 인증 및 네트워크 접근 제어
- [ ] Config Repo 원격 저장소를 사내 Git으로 이전 + 접근 제어
- [ ] Developer Portal CI 파이프라인 구성 (Node 캐시 + Gradle build) → 현재 수동 빌드
- [ ] Observability 대시보드/알람 정의 검토 (현 상태는 샘플 JSON)

필요 시 `HANDOVER_v2.md`를 지속 업데이트하면서 변경 이력(날짜/담당자)을 최하단에 추가해 주세요.
