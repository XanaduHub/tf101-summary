> [CloudNet@] 테라폼 기초 입문 스터디 정리
> 참고 : Terraform Up & Running 2nd Edition

## 테라폼 상태관리

생성한 인프라에 대한 정보를 상태파일에 기록

- `terraform.tfstate` 파일로 기록



## 테라폼 상태 공유 방법

### 테라폼의 원격 백엔드 사용

* `.tfstate` 파일을 원격지에 두고 관리

- (예) AWS S3와 DynamoDB의 결합을 이용

# 테라폼 상태 관리

1. 테라폼 `Workspace` 
   - 다수/분리된/이름이 지정된 워크스페이스 사용, 상태파일 격리
2. 분리된 파일 레이아웃 지정
   - DEV, STG, PRD 환경에 대한 분리, 실수 방지



## 상태관리 (1): Workspace 설정

* 기본적으로 `default` 라는 워크스페이스 사용

* 새 작업공간을 만들기 위해서는 `terraform workspace` 사용

* 워크스페이스 변경?

  - 상태 파일이 저장된 경로 변경

  - 배포되어 있는 인프라에 영향을 주지 않고 테라폼 모듈 테스트 가능

  - 새로운 워크스페이스 생성하여 완전히 동일한 인프라의 복사본을 배포 가능, 하지만 상태 정보는 별도 파일에 저장

* 단점

  1. 모든 작업 공간의 상태 파일은 동일한 백엔드(예. 동일한 S3 버킷)에 저장 됨, 
     * 모든 작업 공간이 동일한 인증과 접근 통제 사용

  2. 코드나 터미널에 현재 작업 공간에 대한 정보 표시 안됨
     * 인프라 파악 및 유지 보수 어려움

  3. 위와 결함된 문제 (잘못된 환경에서 `terraform destroy` 실행)



## 상태관리 (2): 파일 레이아웃을 이용한 구성파일 격리

- 테라폼 프로젝트 생성, 파일레이아웃 구성
  - 각 구성파일을 분리된 폴더 저장 (DEV, STG, PRD ...)
  - 필요에 따라 디렉토리 별로에 서로 다른 백엔드 환경 구성

* 참고

```bash
.
├── global # S3, IAM과 같이 모든 환경에서 사용되는 리소스를 배치
│   └── s3
│       ├── main.tf
│       └── outputs.tf
├── mgmt # 베스천 호스트(Bastion Host), 젠킨스(Jenkins) 와 같은 데브옵스 도구 환경
│   ├── services    # 해당 환경에서 서비스되는 애플리케이션, 각 앱은 자체 폴더에 위치하여 다른 앱과 분리
│   └── vpc         # 해당 환경을 위한 네트워크 토폴로지
├── prod # 사용자용 맵 같은 프로덕션 워크로드 환경
│   ├── services
│   └── vpc
└── stage # 테스트 환경과 같은 사전 프로덕션 워크로드 workload 환경
    ├── data-stores # 해당 환경 별 데이터 저장소. 각 데이터 저장소 역시 자체 폴더에 위치하여 다른 데이터 저장소와 분리
    │   └── mysql
    │       ├── main-vpgsg.tf
    │       ├── main.tf
    │       ├── outputs.tf
    │       ├── terraform.tfstate
    │       └── variables.tf
    └── services
        └── webserver-cluster
            ├── main.tf
            └── user-data.sh
```

