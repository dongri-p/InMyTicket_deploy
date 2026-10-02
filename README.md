# InMyTicket Deploy

실시간 티켓 예매 서비스 **InMyTicket**의 배포 구성입니다. 백엔드·프론트엔드·MySQL을 Docker Compose로 한 번에 띄웁니다.
프로젝트 전체 소개와 배포 과정 트러블슈팅은 [백엔드 저장소 README](https://github.com/dongri-p/InMyTicket_JPA)에 정리되어 있습니다.

* **데모:** https://inmyticket.duckdns.org

## 구성
| 서비스 | 이미지 / 빌드 | 외부 포트 | 역할 |
| --- | --- | --- | --- |
| `frontend` | `../frontend` 빌드 (Nginx) | 80, 443 | React 정적 파일 서빙, HTTPS 종료, `/api/` → `backend:8080` 리버스 프록시 |
| `backend` | `../backend` 빌드 (Spring Boot) | 없음 | REST API (`prod` 프로필) |
| `db` | `mysql:8.0` | 없음 | 데이터 저장 (`db_data` 볼륨) |

* `nginx/default.conf`: 80 요청을 HTTPS 도메인으로 301 리다이렉트하고, 443에서 정적 파일 서빙과 API 프록시를 처리합니다. 컨테이너의 기본 설정을 볼륨으로 덮어씁니다.
* 인증서는 EC2 호스트에서 certbot(`--standalone`)으로 발급·자동 갱신하며, `/etc/letsencrypt`를 읽기 전용으로 마운트합니다.

## 디렉터리 배치
compose가 `../backend`, `../frontend`를 빌드 컨텍스트로 쓰므로 세 저장소를 같은 상위 폴더에 클론합니다.

```text
InMyTicket/
├── backend/            # InMyTicket_JPA
├── frontend/           # InMyTicket_JPA_frontend
└── InMyTicket_deploy/  # 이 저장소
```

## 실행
```bash
cp .env.example .env   # 값 채우기 (.env는 커밋하지 않음)
docker compose up --build -d
```

| 변수 | 설명 |
| --- | --- |
| `DB_PASSWORD`, `DB_ROOT_PASSWORD` | MySQL 계정 비밀번호 |
| `JWT_SECRET` | JWT 서명 키 |
| `ADMIN_LOGIN_ID`, `ADMIN_PASSWORD` | 기동 시 생성/동기화되는 관리자 계정 |
| `KOPIS_API_KEY` | KOPIS 공연 정보 Open API 키 |
| `CORS_ALLOWED_ORIGINS` | 백엔드 CORS 허용 Origin |
| `VITE_API_BASE_URL` | 프론트 빌드 시 들어가는 API 주소 |

## 운영 메모
* 백엔드는 GitHub Actions가 GHCR(`ghcr.io/dongri-p/inmyticket-backend`)에 올린 이미지를 사용합니다. 수동 반영: `docker compose pull backend && docker compose up -d --no-deps backend`
* 프론트만 바꿨을 때: `docker compose build frontend && docker compose up -d --no-deps frontend` (`depends_on` 때문에 그냥 `up --build frontend`를 하면 백엔드까지 빌드됩니다)
* 백엔드 컨테이너를 재생성한 뒤 API가 502로 실패하면 `frontend`를 재시작합니다. Nginx가 `backend` 호스트명을 기동 시점에 한 번만 조회하기 때문입니다.
* 관리자 비밀번호 변경: `.env`의 `ADMIN_PASSWORD`를 수정한 뒤 `docker compose up -d --no-deps backend`
