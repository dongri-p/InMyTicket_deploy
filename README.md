# InMyTicket Deploy

실시간 티켓 예매 서비스 **InMyTicket**의 배포 구성입니다. 백엔드·프론트엔드·MySQL을 Docker Compose로 한 번에 띄웁니다.
프로젝트 전체 소개와 배포 과정 트러블슈팅은 [백엔드 저장소 README](https://github.com/dongri-p/InMyTicket_JPA)에 정리되어 있습니다.

* **데모:** https://inmyticket.duckdns.org

## 구성
| 서비스 | 이미지 / 빌드 | 외부 포트 | 역할 |
| --- | --- | --- | --- |
| `frontend` | `ghcr.io/dongri-p/inmyticket-frontend` (Nginx) | 80, 443 | React 정적 파일 서빙, HTTPS 종료, `/api/` → `backend:8080` 리버스 프록시 |
| `backend` | `ghcr.io/dongri-p/inmyticket-backend` (Spring Boot) | 없음 | REST API (`prod` 프로필) |
| `db` | `mysql:8.0` | 없음 | 데이터 저장 (`db_data` 볼륨) |

* `nginx/default.conf`: 80 요청을 HTTPS 도메인으로 301 리다이렉트하고, 443에서 정적 파일 서빙과 API 프록시를 처리합니다. 컨테이너의 기본 설정을 볼륨으로 덮어씁니다.
* 인증서는 EC2 호스트에서 certbot(`--standalone`)으로 발급·자동 갱신하며, `/etc/letsencrypt`를 읽기 전용으로 마운트합니다.

* 백엔드·프론트엔드 이미지는 각 저장소의 GitHub Actions가 빌드해 GHCR에 올립니다. EC2는 빌드하지 않고 이미지를 받아서 실행만 합니다.

## 실행
```bash
cp .env.example .env   # 값 채우기 (.env는 커밋하지 않음)
docker compose up -d
```

| 변수 | 설명 |
| --- | --- |
| `DB_PASSWORD`, `DB_ROOT_PASSWORD` | MySQL 계정 비밀번호 |
| `JWT_SECRET` | JWT 서명 키 |
| `ADMIN_LOGIN_ID`, `ADMIN_PASSWORD` | 기동 시 생성/동기화되는 관리자 계정 |
| `KOPIS_API_KEY` | KOPIS 공연 정보 Open API 키 |
| `CORS_ALLOWED_ORIGINS` | 백엔드 CORS 허용 Origin |

## 운영 메모
* 배포는 Actions가 자동으로 하지만, 수동으로 반영할 때는 `docker compose pull <서비스> && docker compose up -d --no-deps <서비스>`
* Nginx는 도커 내장 DNS(`resolver 127.0.0.11`)로 `backend`를 요청 시점에 조회하므로, 백엔드 컨테이너를 재생성해도 `frontend` 재시작이 필요 없습니다. `nginx/default.conf`를 고친 뒤에는 `docker compose exec frontend nginx -s reload`로 반영합니다.
* 관리자 비밀번호 변경: `.env`의 `ADMIN_PASSWORD`를 수정한 뒤 `docker compose up -d --no-deps backend`
