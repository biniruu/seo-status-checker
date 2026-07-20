# SEO Status Checker — n8n 워크플로우 + GitHub Actions 자동 실행

웹사이트의 SEO 상태(메타태그·robots.txt·사이트맵)를 매주 자동 점검하고, 판정 결과를 엑셀 리포트로 생성해 이메일로 발송하는 자동화 프로젝트입니다.

실무(웹 프론트엔드 개발)에서 배포·이슈가 있을 때마다 사람이 직접 반복하던 SEO 점검 업무를, 두 단계에 걸쳐 자동화했습니다.

- **1단계 — 로컬 자동화**: n8n(워크플로우 자동화 도구, Docker 셀프호스팅)으로 점검 워크플로우를 직접 설계·구축했습니다.
- **2단계 — CI 확장 (이 저장소)**: 같은 워크플로우를 GitHub Actions에서 실행되도록 확장해, 로컬 PC가 꺼져 있어도 매주 정해진 시각에 점검이 돌고 리포트가 이메일로 도착하는 구조로 전환했습니다.

## 동작 구조

```
GitHub Actions (매주 월 09:00 KST cron / 수동 실행)
  └─ Docker로 n8n 실행
       ├─ n8n import:workflow  ← workflow/seo-status-checker.json
       └─ n8n execute
            └─ [워크플로우] URL 목록 → HTML 수집 → 메타태그 추출
                → 판정 규칙(JS) → robots.txt·Sitemap 점검
                → 엑셀 리포트 생성 → /files 에 저장
  └─ 생성된 리포트를 Artifacts 보관 + 이메일 첨부 발송 (SMTP)
```

1단계(로컬)에서는 n8n의 Schedule Trigger가 스케줄을 담당했지만, 2단계(CI)에서는 **스케줄링 주체가 GitHub Actions의 cron으로 이동**합니다. n8n CLI 실행의 진입점으로는 Manual Trigger를 함께 두었습니다.

## 점검 항목

| 구분 | 항목 | 판정 기준 |
|---|---|---|
| 페이지 단위 | title / description | 존재 여부, 권장 길이 |
| 페이지 단위 | canonical / robots(noindex) / og:title / og:image | 존재 여부, 의도치 않은 색인 차단 탐지 |
| 사이트 단위 | robots.txt | 접근 가능 여부 (응답 내용 형식으로 판정) |
| 사이트 단위 | Sitemap | robots.txt 내 선언 존재 여부 |

판정 결과는 항목별 문제/경고/OK 3단계로 기록되며, 사이트별 한 행씩 엑셀 리포트에 담깁니다.

## 실행 방법

- **자동**: 매주 월요일 09:00 KST(00:00 UTC)에 cron으로 실행됩니다.
- **수동**: Actions 탭 → `SEO Status Check` → `Run workflow`.

실행이 끝나면 (1) 해당 실행의 Artifacts에서 리포트를 내려받을 수 있고, (2) 지정된 주소로 리포트가 첨부된 메일이 발송됩니다.

## 설정 (Secrets)

이메일 발송을 위해 저장소 Settings → Secrets and variables → Actions에 아래 3개를 등록합니다. 자격증명은 코드에 두지 않고 GitHub Secrets로만 관리합니다.

| Secret | 값 |
|---|---|
| `MAIL_USERNAME` | 발신 Gmail 주소 |
| `MAIL_PASSWORD` | Gmail 앱 비밀번호 (2단계 인증 계정에서 발급) |
| `MAIL_TO` | 리포트 수신 주소 |

## 저장소 구성

```
.github/workflows/seo-check.yml   # Actions 워크플로우 (cron + 수동 실행, Docker n8n, 메일 발송)
workflow/seo-status-checker.json  # n8n 워크플로우 (노드 10개)
```

## 구축 중 해결한 문제 (CI 확장 단계)

- **CLI 실행 진입점**: 현재 n8n CLI의 `execute` 명령은 Schedule Trigger만 있는 워크플로우를 실행하지 못합니다("Missing node to start execution"). Schedule Trigger는 유지한 채 Manual Trigger를 나란히 추가해 CLI 진입점을 확보했습니다.
- **파일 쓰기 차단**: n8n은 보안상 파일 쓰기 가능 경로를 제한합니다. `N8N_RESTRICT_FILE_ACCESS_TO=/files` 환경 변수로 허용 경로를 명시해 해결했습니다.
- **컨테이너 권한**: n8n 컨테이너는 node 사용자(uid 1000)로 동작하므로, 러너가 만든 출력 폴더에 쓰기 권한을 부여해야 합니다.

1단계(로컬 구축)에서의 문제 해결 기록은 별도의 시나리오 설계서에 정리되어 있습니다.

## 데모 범위

점검 대상은 공개 사이트 2곳(wikipedia.org, developer.mozilla.org)이며, 대상 사이트의 robots 정책을 존중해 식별 가능한 User-Agent를 명시하고 요청합니다.
