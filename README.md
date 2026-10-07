# InMyTicket Deploy

티켓 예매 사이트 InMyTicket을 EC2 한 대에 띄우는 배포 설정입니다. 백엔드, 프론트엔드, MySQL을 Docker Compose로 한 번에 실행합니다.
프로젝트 소개는 [백엔드 README](https://github.com/dongri-p/InMyTicket_JPA)에 있고, 배포하면서 겪은 문제는 [배포 트러블슈팅 문서](https://github.com/dongri-p/InMyTicket_JPA/blob/main/docs/deployment-troubleshooting.md)에 정리했습니다.

**데모: https://inmyticket.duckdns.org**

## 구성

| 서비스 | 이미지 | 외부 포트 | 하는 일 |
| --- | --- | --- | --- |
| `frontend` | `ghcr.io/dongri-p/inmyticket-frontend` | 80, 443 | React 정적 파일 서빙, HTTPS, `/api/` 요청을 백엔드로 전달 |
| `backend` | `ghcr.io/dongri-p/inmyticket-backend` | 없음 | Spring Boot API (`prod` 프로필) |
| `db` | `mysql:8.0` | 없음 | 데이터 저장 (`db_data` 볼륨) |

- 밖에서 들어올 수 있는 건 nginx(80/443)뿐이고, 백엔드와 DB는 포트를 열지 않았습니다.
- 백엔드·프론트 이미지는 각 저장소의 GitHub Actions가 빌드해서 GHCR에 올립니다. EC2는 빌드하지 않고 이미지를 받아서 실행만 합니다. RAM 1GB 인스턴스에서 직접 빌드하다 메모리가 부족했던 게 이유였습니다.
- `nginx/default.conf`는 80으로 들어온 요청을 HTTPS 도메인으로 보내고, 443에서 화면과 API 프록시를 처리합니다. 컨테이너 기본 설정을 볼륨으로 덮어씁니다.
- 인증서는 EC2 호스트의 certbot이 발급하고 자동 갱신합니다. 컨테이너에는 `/etc/letsencrypt`를 읽기 전용으로 마운트합니다.
- 비밀번호나 키 같은 값은 저장소에 올리지 않고 EC2의 `.env`에만 둡니다.

## 실행

```bash
cp .env.example .env   # 값 채우기 (.env는 커밋하지 않음)
docker compose up -d
```

| 변수 | 설명 |
| --- | --- |
| `DB_PASSWORD`, `DB_ROOT_PASSWORD` | MySQL 비밀번호 |
| `JWT_SECRET` | JWT 서명 키 |
| `ADMIN_LOGIN_ID`, `ADMIN_PASSWORD` | 기동할 때 만들어지는 관리자 계정 |
| `KOPIS_API_KEY` | KOPIS 공연 정보 API 키 |
| `CORS_ALLOWED_ORIGINS` | 백엔드 CORS 허용 주소 |

## 운영하면서 쓰는 명령

- 배포는 Actions가 자동으로 합니다. 손으로 반영할 때는 `docker compose pull <서비스> && docker compose up -d --no-deps <서비스>`를 씁니다.
- nginx는 도커 내장 DNS(`resolver 127.0.0.11`)로 `backend` 주소를 요청할 때마다 다시 찾습니다. 그래서 백엔드를 다시 띄워도 frontend를 재시작할 필요가 없습니다. `nginx/default.conf`를 고친 뒤에는 `docker compose exec frontend nginx -s reload`로 반영합니다.
- 관리자 비밀번호를 바꾸려면 `.env`의 `ADMIN_PASSWORD`를 고치고 `docker compose up -d --no-deps backend`를 실행합니다.

## 아직 부족한 점

- 백엔드를 다시 띄우면 기동하는 20초 정도 API가 502를 반환합니다. healthcheck를 붙여 기동이 끝난 뒤 넘기는 방식으로 줄일 수 있을 것 같습니다.
- DB 백업은 도커 볼륨뿐이라 인스턴스나 볼륨이 망가지면 복구할 방법이 없습니다. 주기적인 `mysqldump`나 EBS 스냅샷이 필요합니다.
- 로그는 `docker logs`로 직접 보는 정도이고, 모니터링이나 알림은 없습니다.
- 서버가 한 대라 이 인스턴스가 죽으면 서비스 전체가 멈춥니다.
