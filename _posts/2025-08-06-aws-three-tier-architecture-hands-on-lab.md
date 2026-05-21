---
title: "AWS 3-Tier 아키텍처 Hands-On Lab"
date: 2025-08-06 13:19:00 +0900
categories: [Cloud, AWS]
tags: [aws, three-tier, vpc, ec2, rds, hands-on]
description: "AWS 친숙도 향상 및 고가용성 설계를 위한 3-Tier 아키텍처 Hands-On 실습"
toc: true
---

## Overview
클라우드 컴퓨팅이 보편화되면서 많은 기업과 엔지니어들이 물리적인 서버 대신 AWS와 같은 클라우드 플랫폼을 활용해 애플리케이션을 구축하고 있습니다. 이때 가장 널리 사용되는 아키텍처 중 하나가 바로 **3-Tier 아키텍처**입니다.

3-Tier 아키텍처는 애플리케이션을 **웹 계층(Web Tier), 애플리케이션 계층(Application Tier), 데이터 계층(Database Tier)로 분리**하여 구성하는 구조입니다. 이러한 구조는 애플리케이션의 **유지보수성**, **확장성**, 그리고 **가용성**을 높이는 데에 매우 효과적입니다.

## VPC 구성

AWS 환경에서 클라우드 인프라를 구성하려면 가장 먼저 VPC 및 서브넷을 구성해야 합니다.
이는 AWS 상에서 우리의 인프라가 동작할 **독립된 네트워크 공간을 정의하는 과정**으로, 모든 리소스의 **보안**, **접근 제어**, **통신 경로**에 직접적인 영향을 줍니다.

VPC를 구성하는 이유는 다음과 같습니다.

- **보안 격리**: VPC는 AWS 내에서 논리적으로 격리된 네트워크 환경을 제공합니다. 외부에서의 접근을 통제하고, 내부 리소스 간 통신을 세밀하게 제어할 수 있습니다.
- **네트워크 제어**: IP 주소 범위, 라우팅 테이블, 인터넷 게이트웨이, NAT 게이트웨이 등을 직접 정의할 수 있어 네트워크 구성의 유연성이 높습니다.
- **확장성과 통합**: VPC는 퍼블릭 서브넷과 프라이빗 서브넷을 조합해 외부 서비스와 내부 시스템을 구분할 수 있으며, VPN, Direct Connect 등을 통해 온프레미스 환경과도 통합할 수 있습니다.
- **운영 효율성**: 인프라의 구성과 확장을 체계적으로 관리할 수 있어, 보안성과 운영 효율성을 동시에 확보할 수 있습니다.

이번 VPC 구성 실습에서는 하나의 VPC를 총 6개의 서브넷으로 나누어 구성 하도록 합니다.

### VPC 및 서브넷 구성 실습

1. AWS에 로그인하여 **VPC>VPC 생성** 화면에서 **새로운 VPC를 생성**합니다. (CIDR 10.0.0.0/16)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image.png)

1. **VPC 설정 편집** 화면에서 **DNS Host Name을 활성화** 합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 1.png)

1. **서브넷>서브넷 생성** 화면에서 아래 표의 정보를 참고하여 **서브넷을 생성**합니다.

| **VPC CIDR** | **서브넷 명(예시)** | **서브넷 CIDR** | **가용 영역** | **설명** |
| --- | --- | --- | --- | --- |
| 10.0.0.0/16 | WEB1 | 10.0.0.0/24 | ap-northeast-2a | Web Server 생성 서브넷 |
| 10.0.0.0/16 | WEB2 | 10.0.1.0/24 | ap-northeast-2c | Web Server 생성 서브넷 |
| 10.0.0.0/16 | APP1 | 10.0.10.0/24 | ap-northeast-2a | Application Server 생성 서브넷 |
| 10.0.0.0/16 | APP2 | 10.0.11.0/24 | ap-northeast-2c | Application Server 생성 서브넷 |
| 10.0.0.0/16 | DB1 | 10.0.20.0/24 | ap-northeast-2a | RDS Database 생성 서브넷 |
| 10.0.0.0/16 | DB2 | 10.0.21.0/24 | ap-northeast-2c | RDS Database 생성 서브넷 |

1. **[새 서브넷 추가]** 기능을 통해 한 화면에서 필요한 서브넷을 모두 생성할 수 있습니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 2.png)

1. 6개의 서브넷 설정을 모두 입력 후 **서브넷을 생성**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 3.png)

---

## 웹 계층(Web-tier) 구성

웹 계층은 **사용자와 애플리케이션이 가장 먼저 만나는 부분**입니다.

웹 브라우저나 모바일 앱을 사용하는 최종 사용자의 요청이 처음 도착하는 곳이며, 사용자가 실제로 보는 화면(UI)을 제공하는 역할을 합니다.

웹 계층의 역할은 다음과 같습니다.

- **사용자 인터페이스 제공:** HTML, CSS, JavaScript, 이미지와 같은 정적 콘텐츠를 제공하거나, 동적 요청을 받아 애플리케이션 계층으로 전달합니다.
- **트래픽 진입점:** 외부 인터넷 트래픽이 VPC 내부로 들어오는 관문 역할을 합니다.
- **보안 경계:** 웹 계층은 일반적으로 **Public Subnet**에 배치되며, 외부 사용자와 직접 통신할 수 있는 유일한 계층이 됩니다. 이 계층 뒤에 있는 애플리케이션/데이터베이스 계층은 외부에서 직접 접근할 수 없습니다.
- **확장성 확보:** Auto Scaling Group과 로드 밸런서를 통해 많은 사용자가 동시에 접속해도 안정적으로 응답할 수 있습니다.

아래 표에 명시된 서비스로 웹 계층을 구성 해보도록 하겠습니다.

| **서비스** | **리소스 명(예시)** | **사용 목적** |
| --- | --- | --- |
| Internet Gateway | ksjeon-igw | 퍼블릭 서브넷에서 외부 인터넷과 통신하기 위한 게이트웨이 |
| NAT Gateway | ksjeon-nat | 프라이빗 서브넷(App 계층 등)의 아웃바운드 인터넷 통신(SNAT) 제공 |
| Route Table | ksjeon-web-rt | VPC 내 서브넷의 라우팅 규칙 및 IGW 연결 설정 |
| ALB | ksjeon-web-alb | 웹 계층 EC2 Auto Scaling Group으로 트래픽 분산 및 로드 밸런싱 |
| EC2 Launch Templates | ksjeon-web-ec2 | 웹 계층 서비스 제공을 위한 웹 서버 인스턴스 생성 템플릿 |
| Auto Scaling Group | ksjeon-web-ec2-asg | 트래픽 부하에 따라 웹 계층 EC2 인스턴스를 자동 확장/축소 |
| Security Group | ksjeon-web-sg | 웹 계층 인스턴스에 대한 네트워크 접근 제어 |
| Target Group | ksjeon-web-alb-tg | ALB가 트래픽을 분산할 대상을 정의하고 헬스 체크 관리 |

### Internet Gateway 구성 실습

1. **VPC**의 **인터넷 게이트웨이** 탭으로 이동합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 4.png)

1. **[인터넷 게이트웨이 생성]** 화면에서 이름 태그를 입력 후 리소스를 생성합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 5.png)

1. 생성된 인터넷 게이트웨이를 **VPC에 연결**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 6.png)

1. 생성된 인터넷 게이트웨이가 올바른 VPC에 연결 되었는지 확인합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 7.png)

### NAT Gateway 구성 실습

1. VPC의 **NAT 게이트웨이** 탭으로 이동합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 8.png)

1. **NAT 게이트웨이 생성** 화면에서 생성 정보를 입력 후 리소스를 생성합니다. NAT 게이트웨이 생성 시 **Elastic IP도 할당** 하도록 합니다. **(각 퍼블릿 서브넷에 2개의 NAT 게이트웨이를 생성)**

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 9.png)

1. 생성한 **NAT 게이트웨이가 모두 프로비저닝 될 때까지** 기다립니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 10.png)

### Route Table 구성 실습

1. VPC의 **라우팅 테이블** 화면으로 이동합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 11.png)

1. **[라우팅 테이블 생성]** 화면에서  생성 정보를 입력 후 라우팅 테이블을 생성합니다. **2개의 퍼블릭 서브넷에 연결할 라우팅 테이블을 모두 생성**하도록 합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 12.png)

1. 생성한 **라우팅 테이블 편집** 화면에서 **인터넷 게이트웨이로의 라우팅 경로**를 잡아줍니다. 

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 13.png)

1. **서브넷 연결 편집** 화면에서 **각각의 퍼블릭 서브넷을 연결** 합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 14.png)

### ALB Security Group 구성 실습

1. **VPC>보안그룹** 화면으로 이동하여 **보안 그룹 생성 버튼**을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 15.png)

1. **인터넷에서 HTTP/80 트래픽을 수신할 수 있도록** 웹 계층 ALB의 보안 그룹 생성 정보를 입력합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 16.png)

### ALB + Target Group 구성 실습

1. **EC2>로드 밸런싱>대상 그룹** 화면으로 이동합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 17.png)

1. 대상 유형을 인스턴스로 선택 후, **대상 그룹 이름을 입력**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 18.png)

1. **대상 그룹에 포함할 인스턴스가 있는 VPC를 선택** 합니다. 다른 설정은 **기본(HTTP)으로 둡니다.**

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 19.png)

1. **상태 검사 경로를 /healthz로 입력** 후 다음 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 20.png)

1. 대상 등록 화면에서 **대상 그룹 생성 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 21.png)

1. **EC2>로드 밸런싱>로드 밸런서** 화면으로 이동합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 22.png)

1. 로드 밸런서 선택 화면에서 **Application Load Balancer의 생성 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 23.png)

1. ALB의 기본 구성은 **인터넷 경계 목적의  IPv4 주소 유형**으로 설정합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 24.png)

1. VPC를 설정 후 앞서 생성한 각각의 **퍼블릭 서브넷에 로드 밸런서를 배치**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 25.png)

1. 앞서 생성한 **보안 그룹과 타겟 그룹을 활용**하여 보안 및 리스너 설정을 완료합니다. (이번 실습에서는 리스너 80으로 테스트를 진행)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 26.png)

1. 구성 정보를 확인 후 **로드 밸런서를 생성**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 27.png)

### EC2 Launch Templates + EC2 Security Group 구성 실습

1. **EC2>인스턴스>시작 템플릿 화면**으로 이동합니다. **[시작 템플릿 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 28.png)

1. **시작 템플릿의 이름을 적고 Auto Scaling 지침을 선택**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 29.png)

1. OS 이미지는 **Amazon Linux 2023 kernel-6.1 AMI**를 선택합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 30.png)

1. **인스턴스 유형은 t3.micro로 설정** 후 인스턴스에 로그인 할 키 페어는 생성하지 않습니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 31.png)

1. 네트워크 설정 영역에서 **보안 그룹 생성을 진행**합니다. 인바운드 규칙은 HTTP만 허용을 하고 소스 유형은 Web ALB의 Security Group으로 설정합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 32.png)

1. 고급 네트워크 설정에서 **Public IP 자동 할당 옵션**을 활성화 합니다. (AWS Session Manager 사용)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 33.png)

1. 스토리지 설정은 기본으로 두고 **리소스 태그를 설정** 해줍니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 34.png)

1. **AWS Session Manager 사용을 위한 IAM 인스턴스 프로파일을 설정**합니다.

[2단계: Session Manager의 인스턴스 권한 확인 또는 추가 - AWS Systems Manager](https://docs.aws.amazon.com/ko_kr/systems-manager/latest/userguide/session-manager-getting-started-instance-profile.html)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 35.png)

1. **사용자 데이터 란에 아래와 같이 입력 후 시작 템플릿을 생성**합니다.

```bash
#!/bin/bash

# 0) 헬스체크 파일을 먼저 만들어두기 (설치가 길어져도 빨리 200 응답 가능)
mkdir -p /var/www/html
echo OK > /var/www/html/healthz
chmod 644 /var/www/html/healthz

# 1) httpd 설치
dnf -y install httpd

# 2) Parameter Store에서 App 엔드포인트 조회
APP_ENDPOINT=$(aws ssm get-parameter \
  --name "/dev/web/app_endpoint" \
  --query "Parameter.Value" \
  --output text \
  --region "ap-northeast-2")

echo "APP_ENDPOINT=${APP_ENDPOINT}" >> /etc/environment

# 3) 기본 페이지
cat > /var/www/html/index.html <<'HTML'
<h1>Web Tier OK</h1>
<ul>
  <li><a href="/app">/app (proxy → App)</a></li>
  <li><a href="/app/db">/app/db (App → RDS)</a></li>
</ul>
HTML

# 4) 리버스 프록시
cat > /etc/httpd/conf.d/revproxy.conf <<CONF
ProxyRequests Off
ProxyPreserveHost On

RedirectMatch 301 ^/app$ /app/

ProxyPass        /app/ http://${APP_ENDPOINT}/app/
ProxyPassReverse /app/ http://${APP_ENDPOINT}/app/
CONF

# 6) 기동
apachectl configtest
systemctl enable httpd
systemctl restart httpd
```

### EC2 Auto Scaling Group 구성 실습

**웹 계층(Web tier)의 Auto Scaling Gruop 구성 실습은 애플리케이션 계층(Application Tier)과 데이터베이스 계층(Database Tier)의 구성이 완료된 후에 진행하도록 합니다.**

1. **EC2>Auto Scaling>Auto Scaling 그룹** 화면으로 이동합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 36.png)

1. 시작 템플릿 선택 화면에서 **Auto Scaling Group 이름을 입력 후 생성해둔 시작 템플릿을 선택**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 37.png)

1. 2단계 화면에서 **VPC 설정 후 이전에 생성한 Web1, Web2 서브넷을 선택**하고 [다음] 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 38.png)

1. 3단계 화면에서 로드 밸런싱 옵션은 **[기존 로드 밸런서에 연결]로 설정** 후 생성해둔 웹 계층의 Target Group을 연결 합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 39.png)

1. 나머지 설정은 그대로 두고 **Elastic Load Balancer 상태 확인 켜기 옵션을 체크**한 후 다음 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 40.png)

1. Auto Scaling Group의 **초기 용량 및 최소, 최대 용량을 설정** 해줍니다.(테스트 목적이기 때문에 대상 추적 없이 생성 예정)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 41.png)

1. 나머지 설정은 그대로 두고 **[다음] 버튼**을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 42.png)

1. 5단계 화면에서 **[다음] 버튼을 클릭**합니다. (테스트 용이기 때문에 별도 SNS 알림 없음)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 43.png)

1. **태그 지정 후 [다음] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 44.png)

1. 검토 화면에서 구성 값을 검토 후 **[Auto Scaling 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 45.png)

1. Auto Scaling Group 생성 후 **Web ALB의 DNS로 접근하여 서비스를 확인**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 46.png)

---

## 애플리케이션 계층(Application-Tier) 구성

애플리케이션 계층은 **비즈니스 로직을 처리하는 핵심 계층**입니다.

사용자가 웹 계층을 통해 보낸 요청이 이곳으로 전달되어, 데이터베이스 계층과 상호작용하며 실제 서비스 기능을 수행합니다.

애플리케이션 계층의 역할은 다음과 같습니다.

- **비즈니스 로직 처리:** 사용자 요청을 해석하고, 서비스 고유의 규칙에 따라 연산 및 데이터 처리를 수행합니다. 예를 들어 로그인 검증, 상품 검색, 결제 처리 등이 포함됩니다.
- **웹 계층과 데이터 계층 연결:** 웹 계층에서 전달된 요청을 데이터베이스 계층에 전달하고, 결과를 가공하여 다시 웹 계층으로 반환합니다.
- **보안 분리:** 애플리케이션 계층은 일반적으로 **Private Subnet**에 배치되며, 외부에서 직접 접근할 수 없습니다. 오직 웹 계층이나 내부 관리용 채널을 통해서만 접근이 허용됩니다.
- **확장성과 가용성:** Auto Scaling Group을 통해 인스턴스를 자동으로 늘리거나 줄일 수 있으며, 내부용 Application Load Balancer(ALB)를 통해 트래픽을 균등하게 분산합니다.
- **유연한 아키텍처:** 컨테이너(ECS, EKS) 또는 서버리스(Lambda) 기반으로도 구현 가능하여 다양한 아키텍처 패턴을 지원합니다.

아래 표에 명시된 서비스로 애플리케이션 계층을 구성 해보도록 하겠습니다.

| **서비스** | **리소스 명(예시)** | **사용 목적** |
| --- | --- | --- |
| Route Table | ksjeon-app-rt | VPC 내 서브넷의 라우팅 규칙 및 NAT Gateway 연결 설정 |
| ALB | ksjeon-app-alb | 애플리케이션 계층 EC2 Auto Scaling Group으로 트래픽 분산 및 로드 밸런싱 |
| EC2 Launch Templates | ksjeon-app-ec2 | 애플리케이션 로직을 처리하는 앱 서버 인스턴스 생성 템플릿 |
| Auto Scaling Group | ksjeon-app-ec2-asg | CPU 사용량에 따라 애플리케이션 계층 EC2 인스턴스를 자동 확장/축소 |
| Security Group | ksjeon-app-sg | 애플리케이션 계층 인스턴스에 대한 네트워크 접근 제어 |
| Target Group | ksjeon-app-alb-tg | ALB가 트래픽을 분산할 대상을 정의하고 헬스 체크 관리 |

### Route Table 구성 실습

1. VPC의 **라우팅 테이블** 화면으로 이동하여 **[라우팅 테이블 생성]** 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 47.png)

1. **Route Table 이름과 연결할 VPC를 설정** 해준 후 생성합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 48.png)

1. 생성한 Route Table 화면의 라우팅 탭에서 **[라우팅 편집] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 49.png)

1. **NAT Gateway로의 라우팅 경로**를 설정한 후 변경 사항을 저장합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 50.png)

1. 서브넷 연결 탭의 **명시적 서브넷 연결 항목에서 [서브넷 연결 편집] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 51.png)

1. 애플리케이션 티어의 프라이빗 서브넷을 체크 후 [연결 저장] 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 52.png)

### ALB Security Group 구성 실습

1. VPC의 보안 그룹 화면으로 이동하여 **[보안 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 53.png)

1. 기본 세부 정보를 입력합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 54.png)

1. 인바운드 규칙의 **유형을 사용자 지정 HTTP/80, 소스를 웹 계층 EC2의 Security Group으로 설정** 후 보안 그룹을 생성합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 55.png)

### ALB + Target Group 구성 실습

1. **EC2>로드 밸런싱>대상 그룹 화면으로 이동 후 [대상 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 56.png)

1. 대상 유형은 **인스턴스로 두고**, 대상 그룹 이름을 입력합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 57.png)

1. **프로토콜은 HTTP/8080으로 설정** 후, 대상 그룹의 VPC를 설정합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 58.png)

1. 상태 검사 설정의 경로를 **/health로 변경**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 59.png)

1. 대상 등록 화면에서 별도 설정 없이 **[대상 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 60.png)

1. **EC2>로드 밸런싱>로드밸런서 화면으로 이동 후 [로드 밸런서] 생성 버튼을 클릭**합니다**.**

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 61.png)

1. 로드 밸런서 유형 화면에서 **Application Load Balancer를 선택**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 62.png)

1. 기본 구성을 **내부 로드 밸런서로 설정**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 63.png)

1. VPC 설정 후, **애플리케이션 계층의 가용 영역 및 서브넷을 매핑**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 64.png)

1. 미리 생성한 App ALB의 보안 그룹을 매핑합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 65.png)

1. 리스너 설정은 **HTTP/80으로 설정** 후 라우팅 액션을 **앞서 생성한 대상 그룹에 매핑**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 66.png)

1. 스크롤 바를 내려 **[로드 밸런서 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 67.png)

### EC2 Launch Templates + EC2 Security Group 구성 실습

1. EC2>인스턴스>시작 템플릿 화면으로 이동 후, **[시작 템플릿 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 68.png)

1. 시작 템플릿의 **이름 및 버전 설명을 입력 후 Auto Scaling 지침을 활성화**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 69.png)

1. OS 이미지는 **Amazon Linux 2023 Kernel-6.1 AMI로 설정**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 70.png)

1. 인스턴스 유형 항목에서 **t3.micro로 설정**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 71.png)

1. 네트워크 설정 항목에서 **[보안 그룹 생성]을 선택 후 이름 및 설명을 입력**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 72.png)

1. [보안 그룹 규칙 추가] 클릭 후, 포트 범위에 애플리케이션 서버의 **서비스 포트인 8080 포트를 적고** **App Tier ALB의 보안 그룹을 소스 유형으로 설정**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 73.png)

1. 리소스 태그 설정 후 고급 세부 정보 탭에서 S**ession Manager 접근을 위한 IAM 인스턴스 프로파일을 설정**합니다.

[2단계: Session Manager의 인스턴스 권한 확인 또는 추가 - AWS Systems Manager](https://docs.aws.amazon.com/ko_kr/systems-manager/latest/userguide/session-manager-getting-started-instance-profile.html)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 74.png)

1. **사용자 데이터 란에 아래와 같이 입력 후 시작 템플릿을 생성**합니다.

```bash
#!/bin/bash

# Node.js와 npm 설치 (Amazon Linux 2023 기준)
dnf -y install nodejs npm mariadb105   # mariadb105 = MySQL 클라이언트

# 앱 디렉터리
mkdir -p /opt/app
cd /opt/app

# 의존성 설치
npm init -y
npm install express mysql2

# 앱 코드 작성 (server.js)
cat > /opt/app/server.js << 'JS'
const express = require('express');
const mysql = require('mysql2/promise');

const app = express();

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 5,
  queueLimit: 0
});

app.get('/health', (_, res) => res.status(200).send('OK'));

app.get('/app', (_, res) => {
  res.send('<h1>App Tier (Node.js) OK</h1><a href="/app/db">/app/db</a>');
});

app.get('/app/db', async (_, res) => {
  try {
    const [[alice]] = await pool.query(
      'SELECT id, name, email, created_at FROM users WHERE email = ? LIMIT 1',
      ['alice@example.com']
    );
    res.json({ status: 'ok', user: alice || null });
  } catch (err) {
    console.error(err);
    res.status(500).json({ status: 'error', message: err.message });
  }
});

const port = process.env.PORT || 8080;
app.listen(port, '0.0.0.0', () => console.log(`App listening on ${port}`));
JS

# 1) 파라미터 스토어에서 DB 접속 정보 조회
DB_HOST=$(aws ssm get-parameter --name "/dev/app/dbhost" --region "ap-northeast-2" --query "Parameter.Value" --output text)
DB_USER=$(aws ssm get-parameter --name "/dev/app/dbuser" --region "ap-northeast-2" --query "Parameter.Value" --output text)
DB_PASS=$(aws ssm get-parameter --name "/dev/app/dbpass" --with-decryption --region "ap-northeast-2" --query "Parameter.Value" --output text)
DB_NAME=$(aws ssm get-parameter --name "/dev/app/dbname" --region "ap-northeast-2" --query "Parameter.Value" --output text)

# 2) 환경변수에 저장 (systemd 서비스에서 로드)
echo "DB_HOST=$DB_HOST" >> /etc/environment
echo "DB_USER=$DB_USER" >> /etc/environment
echo "DB_PASS=$DB_PASS" >> /etc/environment
echo "DB_NAME=$DB_NAME" >> /etc/environment

# 3) DB 없으면 생성 (문자셋 권장)
mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" -e \
"CREATE DATABASE IF NOT EXISTS \`$DB_NAME\` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 4) users 테이블 없을 때만 생성 + 시드
TABLE_EXISTS=$(mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" -Nse \
"SELECT COUNT(*) FROM information_schema.tables
 WHERE table_schema='$DB_NAME' AND table_name='users';")

if [ "${TABLE_EXISTS:-0}" -eq 0 ]; then
  mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" <<'SQL'
CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
SQL
fi

# 5) systemd 유닛 작성
cat > /etc/systemd/system/app.service << 'UNIT'
[Unit]
Description=Node.js App Service (App Tier)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/app
ExecStart=/usr/bin/node /opt/app/server.js
Restart=always
RestartSec=3
Environment=PORT=8080
EnvironmentFile=-/etc/environment
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
UNIT

# 6) 서비스 기동
systemctl daemon-reload
systemctl enable app
systemctl restart app

```

### Parameter Store 환경 변수 구성 실습

1. **EC2>로드 밸런싱>로드밸런서** 화면으로 이동하여 **내부** **ALB의 DNS 주소를 복사 후 따로 저장**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 75.png)

1. **Parameter Store 화면**으로 이동하여 **[파라미터 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 76.png)

1. **파라미터 세부 정보를 입력**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 77.png)

1. **값 항목에 복사한 내부 ALB의 DNS 주소를 붙여넣은 후** 파라미터를 생성합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 78.png)

### EC2 Auto Scaling Group 구성 실습

**애플리케이션 계층(Application Tier)의 Auto Scaling Gruop 구성 실습은 데이터베이스 계층(Database Tier)의 구성이 완료된 후에 진행하도록 합니다.**

1. EC2>Auto Scaling 그룹 화면으로 이동 후 **[Auto Scaling 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 79.png)

1. 이름을 입력하고 **시작 템플릿을 설정**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 80.png)

1. VPC를 설정 후, **애플리케이션 티어의 서브넷을 설정**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 81.png)

1. 로드 밸런싱 옵션은 **기존 로드 밸런서에 연결 선택 후 앞서 생성한 애플리케이션 계층의 Target Group을 연결**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 82.png)

1. **ELB 상태 확인 켜기 옵션을 체크** 후 다음 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 83.png)

1. **Auto Scaling Group의 크기를 설정**합니다. (앞서 생성한 웹 계층 ASG와 같이 설정)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 84.png)

1. **[다음] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 85.png)

1. **태그를 추가**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 86.png)

1. 검토 화면에서 **[Auto Scaling 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 87.png)

---

## 데이터베이스 계층(Database-Tier) 구성

데이터베이스 계층은 **데이터를 영구적으로 저장·관리하는 핵심 계층**입니다.

애플리케이션 계층에서 요청한 데이터를 안정적으로 보관하고, 필요할 때 신속하고 정확하게 제공하는 역할을 수행합니다.

데이터베이스 계층의 역할은 다음과 같습니다.

- **데이터 저장 및 관리:** 사용자 계정, 주문 내역, 결제 정보, 로그 기록 등 서비스 운영에 필요한 모든 데이터를 영구적으로 저장하고 관리합니다.
- **데이터 무결성 및 일관성 유지:** 트랜잭션을 통해 데이터가 손실되거나 잘못 반영되지 않도록 보장합니다. 예를 들어, 결제 중 오류가 발생하더라도 데이터가 꼬이지 않도록 관리합니다.
- **보안과 접근 제어:** 데이터베이스 계층은 일반적으로 **Private Subnet**에 배치되며, 외부에서 직접 접근할 수 없습니다. 오직 애플리케이션 계층에서의 요청만 허용되며, IAM·보안 그룹·암호화(KMS 등)를 통해 접근을 철저히 제어합니다.
- **확장성과 가용성:** Amazon RDS, Aurora, DynamoDB 같은 관리형 서비스를 활용해 고가용성(멀티 AZ), 자동 백업, 읽기 복제본(Read Replica) 등을 지원하여 대규모 트래픽에도 안정적인 성능을 제공합니다.
- **분석 및 최적화:** 데이터베이스는 단순 저장소가 아니라, 쿼리 최적화, 인덱싱, 캐싱과 같은 기능을 통해 빠른 응답 속도를 보장하며, BI·분석 워크로드와도 연계될 수 있습니다.

아래 표에 명시된 서비스로 데이터베이스 계층을 구성 해보도록 하겠습니다.

| **서비스** | **리소스 명(예시)** | **사용 목적** |
| --- | --- | --- |
| Route Table | ksjeon-db-rt | VPC 내 DB 서브넷의 라우팅 규칙 설정(Local 라우팅) |
| Security Group | ksjeon-db-sg | RDS 데이터베이스에  대한 네트워크 접근 제어 |
| Amazon RDS | ksjeon-db-rds | 데이터베이스 계층의 서비스를 위한 RDS 인스턴스 |

### Route Table 구성 실습

1. VPC의 **라우팅 테이블** 화면으로 이동하여 **[라우팅 테이블 생성]** 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 88.png)

1. Route Table 생성 정보 입력 후 [라우팅 테이블 생성] 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 89.png)

1. 생성한 Route Table 화면의 서브넷 연결 탭에서 명시적 **서브넷 연결 항목의 [서브넷 연결 편집] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 90.png)

1. 데이터베이스 계층의 서브넷 추가 후 [연결 저장] 버튼을 클릭합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 91.png)

### Database Security Group 구성 실습

1. VPC>보안>보안 그룹 화면으로 이동하여 **[보안 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 92.png)

1. **기본 세부 정보를 입력**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 93.png)

1. 인바운드 규칙 추가 후 **프로토콜 유형을 MYSQL/Aurora로 설정하고 소스 유형을 App Tier EC2의 보안 그룹으로 설정한 후 [보안 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 94.png)

### Amazon RDS 구성 실습

1. Aurora and RDS>서브넷 그룹 화면으로 이동하여 **[DB 서브넷 그룹 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 95.png)

1. 서브넷 그룹 세부 정보를 입력 후 **서브넷 그룹을 생성할 VPC를 선택**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 96.png)

1. 서브넷 추가 탭에서 추가할 **서브넷이 포함된 가용 영역을 선택**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 97.png)

1. 앞서 생성한 **DB 계층의 서브넷을 선택 후 [생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 98.png)

1. Aurora and RDS>데이터베이스 화면으로 이동하여 **[데이터베이스 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 99.png)

1. 엔진 유형을 **MySQL로 설정**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 100.png)

1. 템플릿을 개발/테스트 선택 후 가용성 및 내구성 옵션은 **인스턴스 2개 옵션을 선택**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 101.png)

1. **DB 인스턴스 식별자 및 마스터 사용자명을 입력한 후 자격 증명 관리는 [자체 관리] 옵션을 선택**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 102.png)

1. **마스터 계정의 패스워드를 입력**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 102.png)

1. 인스턴스 및 스토리지 설정은 그대로 두고 **연결 탭에서 DB 서브넷 그룹이 설정되어 있는 것을 확인**할 수 있습니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 103.png)

1. 앞서 생성한 **데이터베이스 보안 그룹을 매핑**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 104.png)

1. **태그를 추가**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 105.png)

1. **[데이터베이스 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 106.png)

### Parameter Store 환경 변수 구성 실습

DB 연결을 위한 파라미터를 생성하기 위해 **아래 표의 예시를 참고**합니다.

해당 실습은 R**DS 인스턴스 생성 후 진행합니다.**

| Key | Value(예시) |
| --- | --- |
| /dev/app/dbhost | ksjeon-db-rds.che0k2ecafex.ap-northeast-2.rds.amazonaws.com (생성된 RDS의 엔드포인트) |
| /dev/app/dbuser | admin (RDS 마스터 계정) |
| /dev/app/dbpass | rkskekfk1234 (RDS 마스터 계정 패스워드) |
| /dev/app/dbname | testdb (EC2 User Data에서 생성할 DB 명) |
1. **AWS System Manager>파라미터 스토어** 화면으로 이동 후 **[파라미터 생성] 버튼을 클릭**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 107.png)

1. 파라미터의 **이름 및 설명을 입력**합니다.

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 108.png)

1. **파라미터의 값을 입력 후 [파라미터 생성] 버튼을 클릭**합니다. (캡처의 경우 dbhost)

![image.png](/assets/img/posts/aws-three-tier-architecture-hands-on-lab/image 109.png)

---
