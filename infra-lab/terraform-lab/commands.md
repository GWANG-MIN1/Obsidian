# Terraform 명령어 레퍼런스

## 초기화 & 버전
```
terraform version                              # 버전 확인
terraform init                                 # 프로바이더·모듈 다운로드, 백엔드 초기화
terraform init -upgrade                        # 버전 제약 안에서 최신으로 갱신
terraform init -reconfigure                    # 백엔드 설정을 새로 적용 (마이그레이션 안 함)
terraform init -migrate-state                  # 백엔드 변경 시 기존 state 이전
terraform init -backend=false                  # 백엔드 없이 프로바이더만 (CI 검증용)
```

## 검증 & 포맷
```
terraform fmt                                  # 현재 디렉터리 포맷 정리
terraform fmt -recursive                       # 하위 디렉터리까지
terraform fmt -check -diff                     # 고치지 않고 검사만 (CI에서 사용)
terraform validate                             # 문법·타입 검증 (클라우드 접근 없음)
terraform console                              # 표현식을 대화형으로 평가
```

## 계획 & 적용
```
terraform plan                                 # 변경 예정 사항 미리보기
terraform plan -out=tfplan                     # 계획을 파일로 저장
terraform plan -var="env=dev"                  # 변수 직접 주입
terraform plan -var-file=envs/dev.tfvars       # 변수 파일 지정
terraform plan -target=aws_instance.web        # 특정 리소스만 (응급용, 남용 금지)
terraform plan -destroy                        # 삭제 계획 미리보기

terraform apply                                # 적용 (승인 프롬프트)
terraform apply tfplan                         # 저장된 계획을 그대로 적용 (CI 표준)
terraform apply -auto-approve                  # 승인 없이 적용 (자동화 전용)
terraform apply -refresh-only                  # 실제 상태만 state에 반영 (드리프트 흡수)
```

## 삭제
```
terraform destroy                              # 관리 중인 리소스 전체 삭제
terraform destroy -target=aws_instance.web     # 일부만 삭제
terraform plan -destroy -out=tfplan            # 삭제 계획을 검토 후 적용
```

## 출력값
```
terraform output                               # 전체 출력값
terraform output vpc_id                        # 특정 출력값
terraform output -raw vpc_id                   # 따옴표 없이 (스크립트용)
terraform output -json                         # JSON 형식
```

## State 조회
```
terraform state list                           # 관리 중인 리소스 주소 목록
terraform state show aws_vpc.main              # 리소스 속성 상세
terraform state pull > state.json              # 원격 state를 내려받기
terraform state push state.json                # state 업로드 (최후 수단)
terraform show                                 # 현재 state를 사람이 읽기 좋게
terraform show -json tfplan                    # 계획을 JSON으로 (정책 검사 입력)
terraform graph                                # 의존성 그래프 (DOT 형식)
```

## State 조작 (주의)
```
terraform state mv aws_vpc.a aws_vpc.b         # state 안에서 주소 이동 (리네임)
terraform state mv module.a module.b           # 모듈 통째로 이동
terraform state rm aws_instance.web            # 관리 대상에서 제외 (실제 리소스는 유지)
terraform state replace-provider A B           # 프로바이더 소스 교체
terraform force-unlock <LOCK_ID>               # 잠금 강제 해제 (정말 아무도 안 쓸 때만)
```

## 임포트 & 리프레시
```
terraform import aws_vpc.main vpc-0abc123      # 기존 리소스를 state로 편입 (레거시 방식)
terraform plan -generate-config-out=gen.tf     # import 블록 기반 설정 자동 생성
terraform apply -refresh-only                  # 실제 상태를 state에 반영
terraform plan -refresh=false                  # 리프레시 생략 (대규모에서 속도 확보)
```

## Workspace
```
terraform workspace list                       # 목록
terraform workspace new dev                    # 생성
terraform workspace select prod                # 전환
terraform workspace show                       # 현재 workspace
terraform workspace delete dev                 # 삭제
```

## 모듈
```
terraform get                                  # 모듈만 내려받기
terraform get -update                          # 모듈 갱신
terraform providers                            # 사용 중인 프로바이더 트리
terraform providers lock -platform=linux_amd64 # 락 파일에 플랫폼 해시 추가
```

## 로그 & 디버깅
```
TF_LOG=DEBUG terraform apply                   # 디버그 로그 (TRACE·DEBUG·INFO·WARN·ERROR)
TF_LOG_PATH=./tf.log TF_LOG=DEBUG terraform plan
terraform apply -parallelism=5                 # 동시 실행 수 조절 (기본 10)
terraform plan -no-color                       # 색상 코드 제거 (로그 저장용)
```

## 자주 쓰는 환경변수
```
TF_VAR_region=ap-northeast-2                   # 변수 region 주입
TF_INPUT=0                                     # 대화형 입력 비활성화 (CI 필수)
TF_IN_AUTOMATION=1                             # 자동화 환경임을 알려 출력 간소화
AWS_PROFILE=dev                                # AWS 자격증명 프로필 선택
AWS_REGION=ap-northeast-2                      # 리전 지정
```

## 정적 분석 (별도 도구)
```
tfsec .                                        # 보안 취약 설정 탐지
checkov -d .                                   # 정책 위반 검사
trivy config .                                 # IaC 미스컨피그 스캔
terraform-docs markdown table . > README.md    # 변수·출력 문서 자동 생성
tflint                                         # 프로바이더별 린트 (잘못된 인스턴스 타입 등)
```

## 자주 겪는 상황
```
# 락이 걸린 채로 프로세스가 죽었을 때
terraform force-unlock <LOCK_ID>

# 이름만 바꾸고 싶은데 destroy/create 가 잡힐 때 → moved 블록 또는
terraform state mv aws_vpc.old aws_vpc.new

# 콘솔에서 수동 변경한 걸 코드 기준으로 되돌릴 때
terraform apply                                # plan 에서 차이를 확인 후 적용

# 코드는 그대로 두고 실제 변경을 인정할 때
terraform apply -refresh-only
```
