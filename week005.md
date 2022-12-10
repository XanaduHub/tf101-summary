> [CloudNet@] 테라폼 기초 입문 스터디 정리
> 참고 : Terraform Up & Running 2nd Edition

# 테라폼 모듈

- Root module : 기본 작업 디렉토리 의 파일에 정의된 리소스
- Child module
- Published module : 공개 또는 비공개 레지스트리에서 모듈 로드



## 모듈 기본

```terraform
module "<NAME>" { #  모듈에 대한 식별자
  source = "<SOURCE>" # 모듈 코드 찾는 경로

  [CONFIG ...]
}
```



## 모듈의 입력값



## 모듈과 지역변수



## 모듈과 출력



## 모듈 사용 시 주의사항



## 모듈 버전관리

