# 공통 개발 Task

Go와 npm 프로젝트의 의존성 검사 및 GitHub 보안 이슈 알림을 공유합니다.
제품 소스나 설치된 의존성을 수정하지 않습니다. 각 소비 저장소의 workflow가
검사 일정·권한·모듈 구성을 결정하고, 이 저장소는 실제 검사와 이슈 보고를 처리합니다.

## 프로젝트에 연결

```sh
git submodule add https://github.com/w00j00ng/tasks.git tasks
```

프로젝트 루트의 `Taskfile.yml`에서 공통 task를 포함합니다.

```yaml
version: '3'
includes:
  shared:
    taskfile: ./tasks/Taskfile.yml
    dir: .
    flatten: true
```

```sh
go install github.com/go-task/task/v3/cmd/task@v3.54.0
go install golang.org/x/vuln/cmd/govulncheck@v1.9.0
task --exit-code security:scan MODULE=helloworld
task --exit-code security:npm
task security:test
```

Go 검사는 `govulncheck -test -show verbose ./...`, `CGO_ENABLED=0`을 사용합니다.
npm 검사는 루트 lockfile과 개발 의존성을 포함하며 high·critical에서 실패합니다.
`npm audit fix`나 패키지 설치는 수행하지 않습니다. 실행 파일이 있는 Go bin 디렉터리를
PATH에 추가하세요. Task v3.54.0, jq 및 POSIX 명령 도구가 필요하고, 실제 이슈 보고에는
GitHub CLI `gh`가 필요합니다. Python은 사용하지 않습니다.

## CI와 알림

소비 저장소의 검사 job 이름은 `dependency-scan (...)` 형식으로 지정합니다.
검사 step에 `id: scan`을 주고 `GITHUB_OUTPUT`을 전달하면, 확인된 취약점만
`vulnerable=true`를 기록합니다. 이어서 아래 step으로 CI 실패를 유지합니다.

```yaml
- name: Fail for reachable vulnerabilities
  if: steps.scan.outputs.vulnerable == 'true'
  run: exit 1
```

이 step 이름은 보고 task가 취약점 실패를 식별하는 공통 규약입니다. npm에서도 같은
이름을 사용합니다. 일반 설치·레지스트리·네트워크·분석 오류는 발견으로 표시하지 않습니다.
Go는 종료 코드 3, npm은 유효한 audit JSON의 high·critical 개수를 사용해 구분합니다.

보고 job은 전체 검사가 끝난 뒤 `task security:report`를 호출합니다. workflow에서
`schedule`, `refs/heads/main`, 검사 실패 조건을 적용하고, job에 `contents: read`,
`actions: read`, `issues: write`를 부여하세요. `GH_TOKEN: ${{ github.token }}`을 전달하고
`dependency-security-issue` 동시 실행 그룹에서 `cancel-in-progress: false`와 `queue: max`로
직렬화합니다. 대기 중인 보고는 최대 100건까지 보존합니다.
workflow 수준에서도 정기 검사와 push·수동 실행의 동시 실행 그룹을 분리해 정기 보고가
취소되지 않게 하세요. 정기 실행은 workflow 그룹에 `github.run_id`를 사용해 과거 정기
실행의 재시도와도 분리합니다. workflow의 정기 일정은 default branch에 병합한 뒤 동작합니다.

보고 task는 실제 실행 메타데이터도 확인하고, 같은 workflow 실행에서 모듈별로 마지막에
실행한 검사 결과를 읽습니다. 보고 job만 재실행해도 앞선 시도의 검사 결과로 보고를 복구합니다.
이슈는 현재 소비 저장소 안에서 `github-actions[bot]` 작성자와
`<!-- dependency-security-issue:v1 -->` 마커로 찾습니다.

- 열린 이슈가 있으면 본문을 최신 결과로 갱신합니다. 실행 번호·시도 번호 마커로 순서를
  비교하며 이전 실행이나 시도의 늦은 보고는 본문과 댓글을 갱신하지 않습니다.
  기존 이슈에 순서 마커가 없으면 첫 갱신에서 추가합니다.
- 열린 이슈의 미해결 취약점이 다음 정기 검사에서도 발견되면 본문을 갱신하고 날짜별로 다시 알립니다.
- 닫힌 이슈는 보존합니다. 이후 새 검사에서 취약점이 발견되면 새 이슈를 만들며,
  동일한 취약점이 계속 발견되는 경우도 포함합니다.
- 닫힌 이슈의 실행·시도 마커와 실제 취약점 검사 시도를 비교해 과거 보고와 보고 job만의
  재시도는 건너뜁니다. 모든 닫힌 이슈에서 검사 완료 시각과 이슈 종료 시각도 비교하여,
  종료 이전이나 같은 시각에 완료된 결과의 지연 보고도 건너뜁니다.
- 본문 갱신과 알림 댓글을 분리하며, 이슈마다 검사 날짜(KST)별 멘션 댓글을 1회 남깁니다.
- 알림 대상은 `@w00j00ng`입니다. 이메일은 GitHub 참여·멘션·댓글 이메일 설정을 따릅니다.
- 정상 검사는 이슈를 자동 종료하지 않습니다. 조치 후 닫아 주세요. 마커는 유지하세요.

취약점 상세는 연결된 검사 로그에서 확인합니다. 보안 보고 job에서는 checkout의
`submodules: true`가 필요하며, 일반 checkout은 공통 task 내용을 가져오지 않습니다.
서브모듈은 소비 저장소에 기록한 커밋으로 고정하며 자동으로 최신 커밋을 따라가지 않습니다.
공통 변경 후 검증한 커밋으로 서브모듈 포인터를 별도로 갱신하세요.

## 검증

```sh
task security:test
```

실제 GitHub 호출이나 메일 발송 없이 생성·갱신·날짜별 중복 방지·닫힌 이슈 보존·
페이지네이션·API 읽기/쓰기 오류·보고 전용 재시도·모듈별 최종 결과·오래된 보고 차단·
기존 이슈 갱신·종료 후 과거 보고 억제·새 검사에서 계속 발견되는 취약점의 재보고·
다른 저장소 격리·Go 종료 코드·npm 심각도/오류를 검증합니다.
소비 저장소에서도 같은 명령을 실행할 수 있습니다. 실제 메일 전달과 대상 플랫폼별
검사 범위는 이 테스트만으로 보장하지 않습니다.
