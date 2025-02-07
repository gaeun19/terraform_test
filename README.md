# terraform_test
테라폼(Terraform) 기초 튜토리얼

https://www.44bits.io/ko/post/terraform_introduction_infrastrucute_as_code

# 들어가며

별거 없음 그냥 읽어보기

# 테라폼 설치

## window

## **1️⃣ Chocolatey를 사용한 설치 (GPT 권장)**

Chocolatey는 Windows에서 인기 있는 패키지 관리자로, Terraform을 쉽게 설치할 수 있습니다.

### **📌 설치 방법**

1. **Chocolatey 설치 (없다면 실행)**
    - **관리자 권한으로 PowerShell을 실행**한 후 다음 명령어 입력:
    
    ```powershell
    Set-ExecutionPolicy Bypass -Scope Process -Force; `
    [System.Net.ServicePointManager]::SecurityProtocol = `
    [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; `
    iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
    ```
    
2. **Terraform 설치**
    
    ```powershell
    choco install terraform -y
    ```
    
3. **설치 확인**
    
    ```powershell
    terraform -version
    ```
    

✅ **Chocolatey를 사용하면 Terraform을 쉽게 업데이트(`choco upgrade terraform -y`)할 수도 있습니다.**

## MAC

안해봐서 모르겠지만 나온거 그대로 따라하면 될듯

# **테라폼을 사용한 웹 애플리케이션 인프라스트럭처 프로비저닝**

## 1단계 - IAM으로 API 키 받기

https://www.44bits.io/ko/post/publishing_and_managing_aws_user_access_key

위 링크 따라서 차분히 하면 됨.

<개념 정리>

IAM : 권한 관리 - 루트 계정을 직접 사용하기 보다 사용자를 두고 그 사용자에 접속하여 사용

API 키 : 각 사용자를 호출하기 위한 키 // csv 파일로 다운 받아야만 함.

자세한 내용은 읽어보기.

API 키 어떻게 사용하는지 찾아봐야함. - 쓸 줄 모르겠으면 찾아보기(GPT or Google)

## 2단계 - HCL로 리소스 정의하고 AWS에 프로비저닝

여기서부터 터미널 사용해야하니까 vs code 사용

### AWS 프로바이더 정의

Q.  `provider.tf` 파일에 있는 KEY  변수들을 .env  파일에서 넣을 수 있을까?

A. 그렇게 하려면 단계가 좀 많이 필요함.

1단계 : .env 파일에 AWS 자격 증명을 저장

2단계 : PowerShell에서 `.env` 파일을 환경 변수로 변환

```powershell
Get-Content .env | ForEach-Object {
    $name, $value = $_ -split '=', 2
    Set-Item -Path "env:TF_VAR_$name" -Value $value
}
```

3단계 : **Terraform에서 변수 선언 (`variables.tf`)**

```powershell
variable "aws_access_key" {}
variable "aws_secret_key" {}
variable "aws_region" {
  default = "ap-northeast-2"
}
```

4단계 : Terraform Provider 설정 (`provider.tf`)

```powershell
provider "aws" {
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
  region     = var.aws_region
}
```

5단계 : **Terraform 실행**

### **첫 번째 이터레이션: EC2 용 SSH 키 페어 정의**

**첫 번째 스텝: HCL 언어로 필요한 리소스를 정의**

```powershell
$ ssh-keygen -t rsa -b 4096 -C "<EMAIL_ADDRESS>" -f "$HOME/.ssh/web_admin" -N ""
```

![image.png](attachment:b83114ac-ed72-41ad-aefc-f7603c227939:image.png)

**다음과 같이 오류 발생 시 해결 방안**

1. ssh 파일 만들기
2. 명령어 수정해서 진행

```powershell
PS C:\Users\gelee\Desktop\AWS\web_infra> mkdir $env:USERPROFILE\.ssh -Force
```

1. 

**두 번째 스텝: 선언한 리소스들이 생성가능한지 계획(Plan)을 확인**

![image.png](attachment:80ce53f5-e77a-4dd0-860f-e3bcba66cd3f:image.png)

⇒ 오류 원인 : [provider.tf](http://provider.tf) 파일에 키값들이 잘못들어가면 이렇게 됨.

⇒ 해결 방안 : [AWS 프로바이더 정의](https://www.notion.so/Terraform-19272dd4233e80669328f618d4bfa1c8?pvs=21)로 다시 가서 .env 파일 다시 설정하거나 공개된 장소에 올릴 생각 없으면 그냥 넣어도 됨.

**세 번째 스텝: 선언된 리소스들을 아마존 웹 서비스에 적용(Apply)**

⇒ 절대 경로를 리눅스 입장에서 작성해줘야지, 안 그러면 오류 파티를 시작할 수 있음.

![image.png](attachment:98fff749-6e82-4b3d-9f0a-ee3944aff25c:image.png)

새로운 계정을 사용했다면, 키페어가 1개로 변한 것을 볼 수 있다.

### **두 번째 이터레이션: SSH 접속 허용을 위한 시큐리티 그룹**

 [`aws_security_group`](https://www.terraform.io/docs/providers/aws/r/security_group.html) : 리소스 | 키페어와 마찬가지로 인스턴스를 정의하는데 필요

 `web_infra.tf` 맨 아래에 추가

```powershell
resource "aws_security_group" "ssh" {
  name = "allow_ssh_from_all"
  description = "Allow SSH port from all"
  ingress {
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

결과

![image.png](attachment:f56cfa69-9d3e-48ee-817a-c326652d72cb:image.png)

### **세 번째 이터레이션: EC2 인스턴스 정의**

web_infra.tf 파일에 추가 ⇒  VPC(Virtual Private Cloud)의 기본(default) 보안그룹 참조

```powershell
data "aws_security_group" "default" {
  name = "default"
}
```

EC2 선언 방법

```powershell
resource "aws_instance" "web" {
  ami = "ami-0a93a08544874b3b7" # amzn2-ami-hvm-2.0.20200207.1-x86_64-gp2
  instance_type = "t2.micro"
  key_name = aws_key_pair.web_admin.key_name # 주석은 파이썬과 동일하게 달 수 있음.
  vpc_security_group_ids = [
    aws_security_group.ssh.id,
    data.aws_security_group.default.id
  ]
}
```

terraform console 명령어로 대화형 콘솔에서 생성된 리소스의 속성을 확인해볼 수 있음. → 여기에서 EC2의 IP를 알 수 있음

이게  terraform console이지, 하나의 EC2에 접근 한 것은 아님. 따라서 EC2에 접근하고 싶다면 ssh 통신 방식으로 접근할 수 있음

### **네 번째 이터레이션: RDS 인스턴스 정의**

RDS 리소스 선언 하는 방법

```powershell
resource "aws_db_instance" "web_db" {
  allocated_storage = 8 # 할당할 용량
  engine = "mysql" # DB 엔진
  engine_version = "5.6.35"
  instance_class = "db.t2.micro" 
  username = "admin"
  password = "<DB_PASSWORD>" # terraform.tfstate에 변수 저장됨.
  skip_final_snapshot = true # 기본값은 false인데, false로 되어있으면 인스턴스 삭제 어려
}
```

![image.png](attachment:dd00b92f-fb2e-487c-b0b8-53089b9d20bd:image.png)

⇒ 오류 원인 : 엔진 버전이 없어지는 경우 다음과 같이 오류 발생

⇒ 해결 방안 : 아래 명령어를 작성하면 사용 가능한 데이터 베이스 엔진 버전을 알 수 있음. 원하는 버전으로 선택해서 작성하면 됨.

```powershell
aws rds describe-db-engine-versions --engine mysql --query "DBEngineVersions[].EngineVersion"

```

![image.png](attachment:2159aed8-32cd-4a42-90be-742baa7478dc:image.png)

**특정 instance_class를 써야만 한다면 아래 명령어로 올바른 버전 맞춰서 사용하기**

```powershell
aws rds describe-orderable-db-instance-options --engine mysql --query "OrderableDBInstanceOptions[*].[DBInstanceClass,EngineVersion]" --output table

```

나의 최종안

```powershell
resource "aws_db_instance" "web_db" {
  allocated_storage    = 20
  engine              = "mysql"
  engine_version      = "8.0.35"   # 최신 AWS 지원 버전으로 변경
  instance_class      = "db.t3.micro"   # 사용 가능한 인스턴스 클래스
  db_name             = "mydatabase"
  username           = "admin"
  password           = "mypassword"
  parameter_group_name = "default.mysql8.0"
  publicly_accessible = false
  skip_final_snapshot = true
}
```

# **프로비저닝된 인프라스트럭처 일괄 종료**

```powershell
$ mv web_infra.tf /tmp/
$ terraform plan
$ terraform plan -destroy
$ terraform destroy # 일괄 종료 방법
```
