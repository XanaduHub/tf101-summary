
> [CloudNet@] 테라폼 기초 입문 스터디 정리
> 참고 : Terraform Up & Running 2nd Edition


# VPC 환경 구축

`vpc.tf`

```bash
# 프로바이더, VPC 정의, VPC 환경 작성
# 서브넷
# IGW - 외부 통신
# RT - 트래픽 흐름 정의

provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_vpc" "zavar-c2-ex01-vpc" {
  cidr_block           = "10.10.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "zavar-c2-ex01-t101-study"
  }
}

resource "aws_subnet" "zavar-c2-ex01-subnet1" {
  vpc_id     = aws_vpc.zavar-c2-ex01-vpc.id
  cidr_block = "10.10.1.0/24"

  availability_zone = "ap-northeast-2a"

  tags = {
    Name = "zavar-c2-ex01-t101-subnet1"
  }
}

resource "aws_subnet" "zavar-c2-ex01-subnet2" {
  vpc_id     = aws_vpc.zavar-c2-ex01-vpc.id
  cidr_block = "10.10.2.0/24"

  availability_zone = "ap-northeast-2c"

  tags = {
    Name = "zavar-c2-ex01-t101-subnet2"
  }
}

resource "aws_internet_gateway" "zavar-c2-ex01-igw" {
  vpc_id = aws_vpc.zavar-c2-ex01-vpc.id

  tags = {
    Name = "zavar-c2-ex01-t101-igw"
  }
}

resource "aws_route_table" "zavar-c2-ex01-rt" {
  vpc_id = aws_vpc.zavar-c2-ex01-vpc.id

  tags = {
    Name = "zavar-c2-ex01-t101-rt"
  }
}

resource "aws_route_table_association" "zavar-c2-ex01-rtassociation1" {
  subnet_id      = aws_subnet.zavar-c2-ex01-subnet1.id
  route_table_id = aws_route_table.zavar-c2-ex01-rt.id
}

resource "aws_route_table_association" "zavar-c2-ex01-rtassociation2" {
  subnet_id      = aws_subnet.zavar-c2-ex01-subnet2.id
  route_table_id = aws_route_table.zavar-c2-ex01-rt.id
}

resource "aws_route" "zavar-c2-ex01-defaultroute" {
  route_table_id         = aws_route_table.zavar-c2-ex01-rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.zavar-c2-ex01-igw.id
}
```

`sg.tf`

```bash
#Security Group - IGW를 통한 ingress, egress 포트 정의

resource "aws_security_group" "zavar-c2-ex01-sg" {
  vpc_id      = aws_vpc.zavar-c2-ex01-vpc.id
  name        = "zavar-c2-ex01 T101 SG"
  description = "zavar-c2-ex01 T101 Study SG"
}

resource "aws_security_group_rule" "zavar-c2-ex01-inbound" {
  type              = "ingress"
  from_port         = 0
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.zavar-c2-ex01-sg.id
}

resource "aws_security_group_rule" "zavar-c2-ex01-outbound" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.zavar-c2-ex01-sg.id
}
```



`ec.tf`

```bash
# AMI

data "aws_ami" "zavar-c2-ex01-amazonlinux2" {
  most_recent = true
  filter {
    name   = "owner-alias"
    values = ["amazon"]
  }

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-ebs"]
  }

  owners = ["amazon"]
}

resource "aws_instance" "zavar-c2-ex01-ec2" {

  depends_on = [
    aws_internet_gateway.zavar-c2-ex01-igw
  ]

  ami                         = data.aws_ami.zavar-c2-ex01-amazonlinux2.id
  associate_public_ip_address = true
  instance_type               = "t2.micro"
  vpc_security_group_ids      = ["${aws_security_group.zavar-c2-ex01-sg.id}"]
  subnet_id                   = aws_subnet.zavar-c2-ex01-subnet1.id

  user_data = <<-EOF
              #!/bin/bash
              wget https://busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-x86_64
              mv busybox-x86_64 busybox
              chmod +x busybox
              RZAZ=$(curl http://169.254.169.254/latest/meta-data/placement/availability-zone-id)
              IID=$(curl 169.254.169.254/latest/meta-data/instance-id)
              LIP=$(curl 169.254.169.254/latest/meta-data/local-ipv4)
              echo "<h1>RegionAz($RZAZ) : Instance ID($IID) : Private IP($LIP) : Web Server</h1>" > index.html
              nohup ./busybox httpd -f -p 80 &
              EOF

  user_data_replace_on_change = true

  tags = {
    Name = "zavar-c2-ex01-instance"
  }
}

output "zavar-c2-ex01-ec2_public_ip" {
  value       = aws_instance.zavar-c2-ex01-ec2.public_ip
  description = "The public IP of this instance"
}
```

# 데이터 소스 블록

* 테라폼을 실행할 때 마다 provider 별로 가져온 읽기 전용 정보

```
data "<PROVIDER>_<TYPE>" "<NAME>" {
  [CONFIG …]
}
```



* 예

```terraform
data "aws_ami" "zavar-c2-ex01-amazonlinux2" {
  most_recent = true
  filter {
    name   = "owner-alias"
    values = ["amazon"]
  }

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-ebs"]
  }

  owners = ["amazon"]
}

resource "aws_instance" "zavar-c2-ex01-ec2" {
  ami                         = data.aws_ami.zavar-c2-ex01-amazonlinux2.id
  ...
}
```

# ALB, ASG 구축

### ASG(Auto Scaling Groups) 

* EC2 인스턴스의 자동적인 스케일링, 관리를 위해 논리적인 그룹 묶음



### ALB(Application Load Balancer)

* 둘 이상의 AZ에서 EC2, 컨테이너, IP 주소 등에 대해 들어오는 트래픽 자동 분산
* 트래픽 분산, 외부 노출 IP 주소 단일화

`vpc.tf`

```bash
# 프로바이더, VPC 정의, VPC 환경 작성
# 서브넷
# IGW - 외부 통신
# RT - 트래픽 흐름 정의

provider "aws" {
  region = "ap-northeast-2"
}

#
#
resource "aws_vpc" "zavar-c2-ex2-vpc" {
  cidr_block           = "10.10.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "zavar-c2-ex2-t101-study"
  }
}

resource "aws_subnet" "zavar-c2-ex2-subnet1" {
  vpc_id     = aws_vpc.zavar-c2-ex2-vpc.id
  cidr_block = "10.10.1.0/24"

  availability_zone = "ap-northeast-2a"

  tags = {
    Name = "zavar-c2-ex2-t101-subnet1"
  }
}

resource "aws_subnet" "zavar-c2-ex2-subnet2" {
  vpc_id     = aws_vpc.zavar-c2-ex2-vpc.id
  cidr_block = "10.10.2.0/24"

  availability_zone = "ap-northeast-2c"

  tags = {
    Name = "zavar-c2-ex2-t101-subnet2"
  }
}

resource "aws_internet_gateway" "zavar-c2-ex2-igw" {
  vpc_id = aws_vpc.zavar-c2-ex2-vpc.id

  tags = {
    Name = "zavar-c2-ex2-t101-igw"
  }
}

resource "aws_route_table" "zavar-c2-ex2-rt" {
  vpc_id = aws_vpc.zavar-c2-ex2-vpc.id

  tags = {
    Name = "zavar-c2-ex2-t101-rt"
  }
}

resource "aws_route_table_association" "zavar-c2-ex2-rtassociation1" {
  subnet_id      = aws_subnet.zavar-c2-ex2-subnet1.id
  route_table_id = aws_route_table.zavar-c2-ex2-rt.id
}

resource "aws_route_table_association" "zavar-c2-ex2-rtassociation2" {
  subnet_id      = aws_subnet.zavar-c2-ex2-subnet2.id
  route_table_id = aws_route_table.zavar-c2-ex2-rt.id
}

resource "aws_route" "zavar-c2-ex2-defaultroute" {
  route_table_id         = aws_route_table.zavar-c2-ex2-rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.zavar-c2-ex2-igw.id
}
```

`sg.tf`

```bash
#Security Group - IGW를 통한 ingress, egress 포트 정의

resource "aws_security_group" "zavar-c2-ex2-sg" {
  vpc_id      = aws_vpc.zavar-c2-ex2-vpc.id
  name        = "zavar-c2-ex2 T101 SG"
  description = "zavar-c2-ex2 T101 Study SG"
}

resource "aws_security_group_rule" "zavar-c2-ex2-inbound" {
  type              = "ingress"
  from_port         = 0
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.zavar-c2-ex2-sg.id
}

resource "aws_security_group_rule" "zavar-c2-ex2-outbound" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.zavar-c2-ex2-sg.id
}
```

`asg.tf`

```bash
# AMI 정보를 가져와 어떻게 사용할 것인가?
# aws_launch_configuration, aws_autoscaling_group 
# 헬스체크

data "aws_ami" "zavar-c2-ex2-amazonlinux2" {
  most_recent = true
  filter {
    name   = "owner-alias"
    values = ["amazon"]
  }

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-ebs"]
  }

  owners = ["amazon"]
}

resource "aws_launch_configuration" "zavar-c2-ex2-launchconfig" {
  name_prefix                 = "zavar-c2-ex2-"
  image_id                    = data.aws_ami.zavar-c2-ex2-amazonlinux2.id
  instance_type               = "t2.micro"
  security_groups             = [aws_security_group.zavar-c2-ex2-sg.id]
  associate_public_ip_address = true

  user_data = <<-EOF
              #!/bin/bash
              wget https://busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-x86_64
              mv busybox-x86_64 busybox
              chmod +x busybox
              RZAZ=$(curl http://169.254.169.254/latest/meta-data/placement/availability-zone-id)
              IID=$(curl 169.254.169.254/latest/meta-data/instance-id)
              LIP=$(curl 169.254.169.254/latest/meta-data/local-ipv4)
              echo "<h1>RegionAz($RZAZ) : Instance ID($IID) : Private IP($LIP) : Web Server</h1>" > index.html
              nohup ./busybox httpd -f -p 80 &
              EOF

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "zavar-c2-ex2-asg" {
  name                 = "zavar-c2-ex2-asg"
  launch_configuration = aws_launch_configuration.zavar-c2-ex2-launchconfig.name
  vpc_zone_identifier = [
    aws_subnet.zavar-c2-ex2-subnet1.id,
    aws_subnet.zavar-c2-ex2-subnet2.id
  ]
  min_size = 3
  max_size = 10

  health_check_type = "ELB"
  target_group_arns = [aws_lb_target_group.zavar-c2-ex2-alb-tg.arn]

  tag {
    key                 = "Name"
    value               = "terraform-asg"
    propagate_at_launch = true
  }
}
```

`alb.tf`

```bash
# ALB 정의
# 리스너
# 리스너 규칙
# 타겟 그룹
# healthcheck

resource "aws_lb" "zavar-c2-ex2-alb" {
  name               = "t101-alb"
  load_balancer_type = "application"
  subnets = [
    aws_subnet.zavar-c2-ex2-subnet1.id,
    aws_subnet.zavar-c2-ex2-subnet2.id
  ]
  security_groups = [aws_security_group.zavar-c2-ex2-sg.id]

  tags = {
    Name = "t101-alb"
  }
}

resource "aws_lb_listener" "zavar-c2-ex2-http-listener" {
  load_balancer_arn = aws_lb.zavar-c2-ex2-alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "fixed-response"

    fixed_response {
      content_type = "text/plain"
      message_body = "404: page not found - T101 study(s3ich4n)"
      status_code  = 404
    }
  }
}

resource "aws_lb_target_group" "zavar-c2-ex2-alb-tg" {
  name     = "t101-alb-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.zavar-c2-ex2-vpc.id

  health_check {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200-299"
    interval            = 5
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener_rule" "zavar-c2-ex2-alb-listener-rule" {
  listener_arn = aws_lb_listener.zavar-c2-ex2-http-listener.arn
  priority     = 100

  condition {
    path_pattern {
      values = ["*"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.zavar-c2-ex2-alb-tg.arn
  }
}

output "zavar-c2-ex2-alb_dns" {
  value       = aws_lb.zavar-c2-ex2-alb.dns_name
  description = "The DNS Address of the ALB"
}
```



