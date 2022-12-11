> [CloudNet@] 테라폼 기초 입문 스터디 정리
> 참고 : Terraform Up & Running 2nd Edition

# 3tier architecture in AWS

## Terraform?

Terraform은 프로덕션 준비 환경을 생성, 관리 및 배포할 수 있는 코드형 오픈 소스 인프라(IAC) 도구입니다. Terraform은 클라우드 API를 선언적 구성 파일로 코드화합니다. Terraform은 기존 서비스 공급자와 맞춤형 사내 솔루션을 모두 관리할 수 있습니다.

이 프로젝트에서는 에서는 Terraform을 사용하여 AWS에 3계층 애플리케이션을 배포합니다.



![img](last.assets/0CqUoNdOwnB0ZKRfK.png) 

# 사전 준비:

- **AWS** & **Terraform**에 대한 기본 지식
- AWS 계정
- **AWS Access** & **Secret Key**



*In this project, I have used some variables also that I will discuss later in this article.*



**Step 1:-** **VPC** 생성

- `vpc.tf` 

```
# Creating VPC
resource "aws_vpc" "demovpc" {
  cidr_block       = "${var.vpc_cidr}"
  instance_tenancy = "default"tags = {
    Name = "Demo VPC"
  }
}
```



**Step 2:-** **Subnet** 생성

- 이 프로젝트에서는 퍼블릭 및 프라이빗 서브넷이 혼합된 프런트 엔드 계층 및 백엔드 계층에 대해 총 6개의 서브넷을 생성합니다.
- `subnet.tf` 

```
# Creating 1st web subnet 
resource "aws_subnet" "public-subnet-1" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "${var.subnet_cidr}"
  map_public_ip_on_launch = true
  availability_zone = "us-east-1a"
tags = {
    Name = "Web Subnet 1"
  }
}
# Creating 2nd web subnet 
resource "aws_subnet" "public-subnet-2" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "${var.subnet1_cidr}"
  map_public_ip_on_launch = true
  availability_zone = "us-east-1b"
tags = {
    Name = "Web Subnet 2"
  }
}
# Creating 1st application subnet 
resource "aws_subnet" "application-subnet-1" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "${var.subnet2_cidr}"
  map_public_ip_on_launch = false
  availability_zone = "us-east-1a"
tags = {
    Name = "Application Subnet 1"
  }
}
# Creating 2nd application subnet 
resource "aws_subnet" "application-subnet-2" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "${var.subnet3_cidr}"
  map_public_ip_on_launch = false
  availability_zone = "us-east-1b"
tags = {
    Name = "Application Subnet 2"
  }
}
# Create Database Private Subnet
resource "aws_subnet" "database-subnet-1" {
  vpc_id            = "${aws_vpc.demovpc.id}"
  cidr_block        = "${var.subnet4_cidr}"
  availability_zone = "us-east-1a"
tags = {
    Name = "Database Subnet 1"
  }
}
# Create Database Private Subnet
resource "aws_subnet" "database-subnet-2" {
  vpc_id            = "${aws_vpc.demovpc.id}"
  cidr_block        = "${var.subnet5_cidr}"
  availability_zone = "us-east-1a"
tags = {
    Name = "Database Subnet 1"
  }
}
```



**Step 3:-** **Internet Gateway** 생성

- `igw.tf` 

```
# Creating Internet Gateway 
resource "aws_internet_gateway" "demogateway" {
  vpc_id = "${aws_vpc.demovpc.id}"
}
```



**Step 4:-**  **Route table**

- `route_table_public.tf`
- 다음 코드에서 새 route table을 만들고 모든 요청을 0.0.0.0/0 CIDR 블록으로 전달합니다.
- 또한 이 route table을 이전에 만든 서브넷에 연결합니다. 따라서 퍼블릭 서브넷으로 작동합니다.

```
# Creating Route Table
resource "aws_route_table" "route" {
    vpc_id = "${aws_vpc.demovpc.id}"
route {
        cidr_block = "0.0.0.0/0"
        gateway_id = "${aws_internet_gateway.demogateway.id}"
    }
tags = {
        Name = "Route to internet"
    }
}
# Associating Route Table
resource "aws_route_table_association" "rt1" {
    subnet_id = "${aws_subnet.demosubnet.id}"
    route_table_id = "${aws_route_table.route.id}"
}
# Associating Route Table
resource "aws_route_table_association" "rt2" {
    subnet_id = "${aws_subnet.demosubnet1.id}"
    route_table_id = "${aws_route_table.route.id}"
}
```



**Step 5:-** **EC2 instances** 생성

- `ec2.tf`
- 다음 코드에서는 userdata를 사용하여 EC2 인스턴스를 구성했습니다. 
-  data.sh 에 대해서는 나주에 설명하겠습니다.

```
# Creating 1st EC2 instance in Public Subnet
resource "aws_instance" "demoinstance" {
  ami                         = "ami-087c17d1fe0178315"
  instance_type               = "t2.micro"
  count                       = 1
  key_name                    = "tests"
  vpc_security_group_ids      = ["${aws_security_group.demosg.id}"]
  subnet_id                   = "${aws_subnet.demoinstance.id}"
  associate_public_ip_address = true
  user_data                   = "${file("data.sh")}"
tags = {
    Name = "My Public Instance"
  }
}
# Creating 2nd EC2 instance in Public Subnet
resource "aws_instance" "demoinstance1" {
  ami                         = "ami-087c17d1fe0178315"
  instance_type               = "t2.micro"
  count                       = 1
  key_name                    = "tests"
  vpc_security_group_ids      = ["${aws_security_group.demosg.id}"]
  subnet_id                   = "${aws_subnet.demoinstance.id}"
  associate_public_ip_address = true
  user_data                   = "${file("data.sh")}"
tags = {
    Name = "My Public Instance 2"
  }
}
```



**Step 6:-** **FrontEnd** **tier**을 위한 **Security Group** 생성

- `web_sg.tf` 
- 인바운드 연결을 위해 80,443 및 22개 포트를 열었고 아웃바운드 연결을 위해 모든 포트를 열었습니다.

```
# Creating Security Group 
resource "aws_security_group" "demosg" {
  vpc_id = "${aws_vpc.demovpc.id}"
# Inbound Rules
  # HTTP access from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
# HTTPS access from anywhere
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
# SSH access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
# Outbound Rules
  # Internet access to anywhere
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
tags = {
    Name = "Web SG"
  }
}
```



**Step 7:-** **Database** **tier**을 위한  **Security Group** 생성

-  `database_sg.tf` 
- 인바운드 연결을 위해 3306 포트를 열었고 아웃바운드 연결을 위해 모든 포트를 열었습니다.

```
# Create Database Security Group
resource "aws_security_group" "database-sg" {
  name        = "Database SG"
  description = "Allow inbound traffic from application layer"
  vpc_id      = aws_vpc.demovpc.id
ingress {
    description     = "Allow traffic from application layer"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.demosg.id]
  }
egress {
    from_port   = 32768
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
tags = {
    Name = "Database SG"
  }
```



**Step 8:-** **Application Load Balancer** 생성

-  `alb.tf` 
