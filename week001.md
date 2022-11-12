> [CloudNet@] 테라폼 기초 입문 스터디 정리
> 참고 : Terraform Up & Running 2nd Edition

# DevOps, IaC, Terraform

* DevOps

> 소프트웨어를 효율적으로 전달하는 프로세스

* IaC(Infrastructure as Code) : 인프라환경을 코드로 작성 -> Terraform이 그중 하나

# Terraform 

HashiCorp 에서 만든 IaC 도구



## IaC 장점

- 자급식 배포 (**Self-service**): 배포 프로세스 자동화, 개발자가 자체적인 인프라 배포 작성 배포
- 속도와 안정성 (**Speed and safety**) : 자동/일관적, human error 발생 가능성 축소
- 문서화 (**Documentation**) : 소스 파일로 인프라 상태 표현
- 버전 관리 (**Version control**) : 인프라의 변경 내용 기록, 원상 복구 가능
- 유효성 검증 (**Validation**) : 인프라 변경시 검증, 자동화된 테스트 가능
- 재사용성 (**Reuse**) : 인프라를 재사용 가능한 모듈로 패키징 가능

# 테라폼



## 테라폼의 기본 개념

- **resource** : 생성할 인프라 자원
- **provider** : 테라폼으로 정의할 Infrastructure Provider(AWS, Microsoft Azure, GCP 등)
- **output** : 인프라를 프로비저닝 한 후에 생성된 자원을 `output` 부분으로 출력 가능
- **backend** : 테라폼 상태 저장 공간, 
- **module** : 공통 부분 정의
- **remote state** : VPC, IAM 등과 같이 여러 서비스가 공통으로 사용하는 것을 사용 가능 



## 테라폼 코드

* HCL(Hashicorp Configuration Language) 로 작성

* `terraform <args>` 형식의 명령어로 실행
  * OS마다 다른 테라폼 바이너리가 `provider`를 대신해 API 호출, 리소스 생성
* 테라폼은 인프라 정보가 담겨 있는 테라폼 구성 파일을 생성하여 API를 호출



## 테라폼 구동 (3단계)

### 테라폼 프로젝트를 initialize

```
terraform init
```

* 지정한 backend에 상태 저장을 위한 `.tfstate` 파일 생성
  * 가장 마지막에 적용한 테라폼 내역이 저장 됨
* 완료 시, local에 `.tfstate`에 정의된 내용을 담은 `.terraform` 파일이 생성 됨
* 기존에 `.tfstate` 존재 시 init작업 통해 local에 sync가능

### 수행 동작 테스트

```
terraform plan
```

* 미리 예측 결과 보여줌 
* `terraform plan` 명령어는 어떠한 형상에도 변화를 주지 않음



### 프로비저닝 수행

```
terraform apply
```

* 실제 인프라 배포
* AWS 상에 실제로 해당 인프라가 생성되고 작업 결과가 backend의 `.tfstate` 파일에 저장 됨
* 결과는 local의 .terraform 파일에도 저장 됨



## `resource` 정의 

* 생성할 인프라 자원에 대해 정의 방법

```Terraform
resource "<PROVIDER>_<TYPE>" "<NAME>" {
  [CONFIG ...]
}
```



* 예

```bash
# CSP 공급자 설정
provider "aws" {
  region = "ap-northeast-2"
}

# 리소스 정의 <PROVIDER>_<TYPE>" "<NAME>
resource "aws_instance" "example" {
  ami           = "ami-0c76973fbe0ee100c"
  instance_type = "t2.micro"

  tags = {
    Name = "t101-study-renamed"
  }
}
```



## 참조(reference) 

* 코드의 다른 부분에서 사전에 정의한 리소스의 특정 값에 액세스

```terraform
<PROVIDER>_<TYPE>.<NAME>.<ATTRIBUTE>
```

* 예

````
aws_security_group.instance.id
````



## 변수(variables) 표현

* 예) 1 - 파일에 분리된 예시코드

`variables.tf`

```
variable "server_port" {
  description = "The port the server will use for HTTP requests"
  type        = number
  default     = 8080
}
```

* 예) 2 -   파일에 작성된 프로바이더, 리소스 정의와 변수 참조

`main.tf` 

```
...

resource "aws_security_group" "instance" {
  name = var.security_group_name # 변수 참조

  ingress {
    from_port   = var.server_port
    to_port     = var.server_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

variable "security_group_name" { # 변수 정의
  description = "The name of the security group"
  type        = string
  default     = "terraform-my-instance"
}

...
```
