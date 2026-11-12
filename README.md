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

Go 검사는 `govulncheck -json -test ./...`를 한 번 실행합니다. 명시한 `CGO_ENABLED`를
상속하며 미설정 시 0을 사용합니다. 실제 `go env`의 GOOS·GOARCH·CGO_ENABLED·GOVERSION을 기록합니다.
JSON 모드의 종료 코드 0도 발견 결과를 포함할 수 있으므로 JSON의 `finding.trace[0]`을 판정합니다.
`function`이 있으면 symbol, `package`만 있으면 package, 둘 다 없으면 module 수준입니다.
같은 OSV·의존 모듈·버전은 가장 강한 수준 하나로 합치고 패키지·호출 경로는 보존합니다.
symbol만 로컬 종료 코드 3으로 실패하며 package/module은 의존성 업데이트 경고로 성공합니다.
빈 JSON·손상된 JSON·source/symbol 설정 또는 SBOM 누락·검사 실행 오류는 정상 결과로 처리하지 않습니다.
npm 검사는 루트 lockfile과 개발 의존성을 포함하며 high·critical에서 실패합니다.
`npm audit fix`나 패키지 설치는 수행하지 않습니다. 실행 파일이 있는 Go bin 디렉터리를
PATH에 추가하세요. Task v3.54.0, jq 및 POSIX 명령 도구가 필요하고, 실제 이슈 보고에는
GitHub CLI `gh`가 필요합니다. Python은 사용하지 않습니다.

## CI와 알림

Go 검사 job 이름은 `dependency-scan (go/<module>/<config>)` 형식으로 지정합니다.
config는 네이티브 runner의 `linux-cgo0`, `linux-cgo1`, `windows-cgo0`, `macos-cgo1` 중 하나입니다.
각 job의 검사 step에는 다음 환경변수를 전달하세요.

```yaml
env:
  SECURITY_RESULT_FILE: ${{ runner.temp }}/security-result.json
  SECURITY_SCAN_KEY: go-${{ matrix.module }}-${{ matrix.target.id }}
```

결과 JSON의 schema_version은 1이며 repository·head_sha·run_id·run_attempt·scan_key·module,
실제 target(goos/goarch/cgo)·go_version·scanner_version·scan_status(complete/error)·scanner_exit와
findings를 포함합니다. 각 finding은 osv·level·module·version·fixed_version·packages·traces를 담습니다.
scanner_exit은 실제 검사 프로세스 종료값이며, 실행 전 Go 환경 조회 실패 시 null입니다.
검사 종료가 0이어도 JSON 파싱 실패는 scan_status=error로 기록하고 작업은 실패합니다.
토큰이나 환경변수 전체를 기록하지 않습니다. 로컬에서 결과 파일을 지정하지 않으면 임시 파일을 정리합니다.
`actions/upload-artifact`를 `if: always()`로 실행해 `security-result.json` 하나를
`security-scan-<scan_key>-attempt-<github.run_attempt>` 이름으로 올리세요. 실패 판정 step보다 먼저 업로드합니다.

검사 step에 `id: scan`을 주고 `GITHUB_OUTPUT`을 전달하면, 확인된 취약점만
`vulnerable=true`를 기록합니다. Go에서는 symbol에만 기록합니다. 이어서 아래 step으로 CI 실패를 유지합니다.

```yaml
- name: Fail for reachable vulnerabilities
  if: steps.scan.outputs.vulnerable == 'true'
  run: exit 1
```

이 step 이름은 npm과 기존 `dependency-scan (...)` job의 실패를 식별하는 공통 규약입니다.
새 Go job은 job 성공 여부와 별개로 결과 산출물을 읽습니다. npm은 유효한 audit JSON의
high·critical 개수를 사용하며 호출 경로 분석으로 해석하지 않습니다. 일반 설치·레지스트리·
네트워크·분석 오류는 발견으로 표시하지 않습니다.

보고 job은 전체 검사가 끝난 뒤 `task security:report`를 호출합니다. workflow에서
`schedule`, `refs/heads/main`에서 검사 성공·실패와 관계없이 실행하고, job에 `contents: read`,
`actions: read`, `issues: write`를 부여하세요. `GH_TOKEN: ${{ github.token }}`을 전달하고
`dependency-security-issue` 동시 실행 그룹에서 `cancel-in-progress: false`와 `queue: max`로
직렬화합니다. 대기 중인 보고는 최대 100건까지 보존합니다.
workflow 수준에서도 정기 검사와 push·수동 실행의 동시 실행 그룹을 분리해 정기 보고가
취소되지 않게 하세요. 정기 실행은 workflow 그룹에 `github.run_id`를 사용해 과거 정기
실행의 재시도와도 분리합니다. workflow의 정기 일정은 default branch에 병합한 뒤 동작합니다.

보고 task는 실제 실행 메타데이터도 확인하고, 같은 저장소·workflow 실행에서 전체 job 이름별
최신 검사 시도의 결과를 읽습니다. 플랫폼·CGO 결과는 서로 대체하지 않습니다. 보고 job만 재실행하면
앞선 최신 검사 산출물을 재사용합니다. 저장소·SHA·run·검사 시도·scan_key·module·실제 target·스키마를 검증하고,
최신 산출물이 누락·만료·손상됐으면 과거 결과로 대체하지 않고 판정 불가로 실패합니다.
API 읽기·다운로드 실패는 이슈를 쓰기 전에 전파합니다. 유효한 발견과 다른 검사 오류가 함께 있으면
발견 결과와 판정 불가 대상을 함께 보고하고 작업은 실패합니다. npm만 검사하는 소비 저장소에는
산출물 API를 요청하지 않습니다. 이슈는 호출 경로가 확인된 Go 결과·의존성 업데이트 경고·npm 결과를 구분합니다.
같은 OSV·의존 모듈·버전·수준은 이슈 한 행에 모으고 영향받은 패키지와 모든 대상·검사 링크를 표시합니다.
보고 단계에서 각 trace 첫 프레임의 가장 강한 근거가 산출물의 선언 수준과 다른 검사도 판정 불가로 처리합니다.
스캔·보고 양쪽에서 trace 필드 타입과 함수→패키지→모듈 관계도 검증합니다.
이슈는 현재 소비 저장소 안에서 `github-actions[bot]` 작성자와
`<!-- dependency-security-issue:v1 -->` 마커로 찾습니다.

- 열린 이슈가 있으면 본문을 최신 결과로 갱신합니다. 실행 번호·시도 번호 마커로 순서를
  비교하며 이전 실행이나 시도의 늦은 보고는 본문과 댓글을 갱신하지 않습니다.
  기존 이슈에 순서 마커가 없으면 첫 갱신에서 추가합니다.
- 열린 이슈의 미해결 취약점이 다음 정기 검사에서도 발견되면 본문을 갱신하고 날짜별로 다시 알립니다.
- 갱신 응답의 이슈 상태가 이미 닫혀 있으면 새 이슈에 보고합니다. 열린 상태의 갱신 응답을
  확인할 때까지 이전 보고의 실행 식별 정보를 숨김 `dependency-security-pending` 마커에 보관하여,
  후속 API 실패 후 재시도에서도 새 결과가 이미 보고된 것으로 처리되지 않게 합니다.
  확인된 갱신은 마커를 제거한 뒤 알림을 남깁니다. 열린 상태로 갱신된 뒤 사용자가 닫으면
  완료된 보고로 처리합니다. 종료와 갱신이 경합하면 닫힌 이슈에 복구 마커가 남을 수 있습니다.
  이전 전체 본문을 담은 마커도 계속 읽습니다.
- 닫힌 이슈는 보존합니다. 이후 새 검사에서 취약점이 발견되면 새 이슈를 만들며,
  동일한 취약점이 계속 발견되는 경우도 포함합니다.
- 이슈 본문에 `<!-- dependency-security-source:SHA:RUN_ID -->`를 기록합니다. 닫힌 이슈와
  커밋 SHA·실행 ID가 같은 재시도는 이미 보고한 검사로 처리합니다. 같은 커밋도 다음 정기 실행의
  ID가 다르면 새 검사로 보고합니다. 검사 완료·이슈 종료 시각은 보고 여부에 사용하지 않습니다.
- 기존 이슈는 실행 순서 마커 또는 본문의 검사 SHA·실행 링크로 재전송을 식별합니다.
  과거 실행 번호의 늦은 보고도 건너뜁니다. 날짜는 일별 멘션 중복 방지에 사용합니다.
- 본문 갱신과 알림 댓글을 분리하며, 이슈마다 검사 날짜(KST)별 멘션 댓글을 1회 남깁니다.
- 알림 대상은 `@w00j00ng`입니다. 이메일은 GitHub 참여·멘션·댓글 이메일 설정을 따릅니다.
- 정상 검사는 이슈를 자동 종료하지 않습니다. 조치 후 닫아 주세요. 마커는 유지하세요.

GitHub 본문 한도를 지키기 위해 보고 본문은 UTF-8 60,000바이트 이내의 완전한 행으로 제한하며,
복구 정보와 안내를 위한 공간을 남깁니다. 생략이 발생하면 본문 상단에 전체 Go 보고 항목 건수를
표시하고 검사 로그 링크를 남깁니다. 각 검사의 전체 JSON 결과는 산출물과 로그에서 확인할 수 있습니다.

취약점 상세는 연결된 검사 로그에서 확인합니다. 보안 보고 job에서는 checkout의
`submodules: true`가 필요하며, 일반 checkout은 공통 task 내용을 가져오지 않습니다.
서브모듈은 소비 저장소에 기록한 커밋으로 고정하며 자동으로 최신 커밋을 따라가지 않습니다.
공통 변경 후 검증한 커밋으로 서브모듈 포인터를 별도로 갱신하세요.

## 검증

테스트 구현은 `Taskfile.test.yml`에 두고 루트 `Taskfile.yml`에서 포함합니다.
공통 저장소와 소비 저장소의 루트에서 `task security:test`로 실행합니다.

```sh
task security:test
```

실제 GitHub 호출이나 메일 발송 없이 생성·갱신·날짜별 중복 방지·닫힌 이슈 보존·
필수 API 페이지네이션 옵션·여러 페이지의 검사/이슈/댓글·API 읽기/쓰기 오류·보고 전용 재시도·모듈별 최종 결과·오래된 보고 차단·
기존 이슈 갱신·종료 후 과거 보고 억제·새 검사에서 계속 발견되는 취약점의 재보고·
보고 중 수동 종료·새 이슈 생성/갱신 확정 실패 후 복구·다른 저장소 격리·Go JSON 수준 판정·
실제 Task의 CGO 상속·산출물 정체성/최신 시도/누락/만료/손상·부분 검사 오류·npm 심각도/오류를 검증합니다.
공통 CI는 Ubuntu·Windows(Bash)·macOS에서 루트와 소비 include 구성을 모두 검사합니다.
소비 저장소에서도 같은 명령을 실행할 수 있습니다. 실제 메일 전달은 이 테스트로 보장하지 않습니다.
