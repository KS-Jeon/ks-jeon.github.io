---
title: "EC2와 S3 Bucket을 활용한 .Net Core Application 구성"
date: 2025-12-05
categories: [Cloud, AWS]
tags: [aws, ec2, s3, dotnet, application]
description: "EC2 가상 머신과 S3 버킷을 활용한 .Net  Core Application 구성 가이드"
toc: true
---

## Introduction

현대의 웹 애플리케이션 운영 환경에서는 다양한 배포 방식과 스토리지 구성 방안이 사용되고 있으며, 이에 따라 안정적이고 효율적인 배포 전략의 중요성이 점차 높아지고 있다. 본 문서는 AWS 환경에서 **Amazon EC2와 Amazon S3를 활용하여 .NET Core 기반 애플리케이션을 수동 배포 방식으로 구성하는 절차와 고려사항**을 중심으로 소개한다.

구조적 이해를 돕기 위해 **SimplCommerce 프로젝트를 샘플 애플리케이션으로 활용하여**, EC2 인스턴스에서 애플리케이션을 실행하고 S3를 정적 콘텐츠 저장소로 연계하는 실제 구성 단계를 중심으로 정리한다. 이번 구성은 초기 구축 단계로서 **수동 배포 방식을 적용**하였으며, 자동화된 배포 파이프라인이나 고도화된 운영 방식은 추후 단계에서 확장할 예정이다.

이를 통해 .NET Core 기반 애플리케이션을 EC2 환경에 안정적으로 배포하고, S3를 활용한 정적 파일 저장 구조를 이해하는 데 필요한 실질적인 정보를 정리한다.

**참고 자료**
- [https://github.com/simplcommerce/simplcommerce](https://github.com/simplcommerce/simplcommerce)

---

## Architecture Overview

본 문서는 Amazon EC2 상에서 **Nginx와 .NET Core 기반 SimplCommerce 애플리케이션(Kestrel)을 함께 구성**하여 단일 인스턴스 형태로 배포하는 방안을 다루고 있다. 또한 애플리케이션에서 업로드되는 이미지 파일은 **Amazon S3에 저장 및 제공**되도록 구성함으로써, EC2 인스턴스의 디스크 사용을 최소화하고 미디어 파일 관리의 효율성을 확보하였다.

테스트 환경은 다음과 같은 단일 EC2 기반 구성으로 구현되었다.

- **Amazon EC2**
    1. **Web Tier**: Nginx Reverse Proxy 및 Cache
    2. **Application Tier**: .NET Core SimplCommerce 실행 환경
    3. **Database Tier**: Docker 기반 SQL Server 실행 환경
- **Amazon S3**
    1. SimplCommerce의 **미디어 파일 저장 및 제공**
    2. EC2 인스턴스에서 업로드된 이미지의 관리
- **S3 Bucket Policy 기반 접근 제어**
    1. 본 문서에서는 EC2 Instance Profile(IAM Role)을 사용하지 않고, **S3 Bucket Policy를 통해 특정 EC2 인스턴스에서만 업로드(PutObject)를 허용하는 방식으로 구성하였다.**
    2. 웹 브라우저가 이미지를 정상적으로 로딩할 수 있도록, **GetObject 요청은 Public Read로 허용**하였다.
    3. 이를 통해 업로드 경로는 Private, 조회 경로는 Public로 명확히 분리하여 **필요한 최소 접근만 허용하는 방식(Least Privilege)**을 유지하였다.
- **수동 배포 방식(Manual Deployment)**
    1. 로컬 환경에서 빌드한 후 EC2로 직접 업로드
    2. systemd 기반 Kestrel 서비스 재시작을 통해 배포 반영

또한 본 문서에서 소개하는 구성은 **초기 도입 및 테스트 환경(PoC)** 에 적합한 구조로, 향후 운영 환경에서는 **서비스 계층을 분리한 Multi-Tier 아키텍처로 확장하는 것이 일반적인 Best Practice**이다.

---

## 배포 환경 정의

이번 장에서는 SimplCommerce 애플리케이션을 배포하기 위해 구성한 **EC2 인스턴스의 사양, 런타임 환경, 보안 설정, 디렉터리 구조**를 정의한다.

### 인스턴스 사양

**테스트 및 초기 PoC 용도**이므로 경량 인스턴스인 `t3.medium` 사양을 사용하였다.
Web(Nginx), App(.NET), DB(SQL Server Docker)의 통합 실행을 고려하여 **4GB RAM 환경에서도 안정적으로 실행 가능한 최소 사양**으로 구성하였다.

| 항목 | 값 |
| --- | --- |
| Instance Type | t3.medium |
| vCPU / Memory | 2 vCPU / 4GB RAM |
| Root Volume | 30GB (gp3) |
| OS Image | Amazon Linux 2023 |
| Availability Zone | ap-northeast-2a |
| Network | Public Subnet, Auto-assign Public IP 활성화 |
| Key Pair | RSA 기반 EC2 SSH Key |
| Instance Profile (IAM Role) | **EC2SessionManagerRole**<br>→ AWS Systems Manager (SSM, Session Manager) 접속 용도<br>→ **S3 접근 권한은 IAM Role이 아닌 버킷 정책(Bucket Policy)에서 전부 제어** |

### 런타임 환경

SimplCommerce는 Kestrel 기반으로 실행되며, Nginx와 조합하여 Reverse Proxy 구조를 구성하였다. 또한 SQL Server는 **Docker 기반으로 경량 구성**하여 단일 인스턴스 내에서 데이터베이스 환경을 간단히 재현할 수 있도록 했다.

| 구성 요소 | 버전 / 설명 |
| --- | --- |
| .NET Runtime | .NET 8 (또는 SimplCommerce 지원 버전 기준) |
| Nginx | Stable Release |
| Docker Engine | 26.x |
| SQL Server | mcr.microsoft.com/mssql/server:2019-latest |
| Systemd 서비스 | SimplCommerce Kestrel Service 구성 |

### 보안 및 네트워크 구성

본 구성에서 EC2 인스턴스는 **외부 서비스 제공을 위해 80(HTTP)과 443(HTTPS) 포트를 개방**하였으며, 초기 설정 및 점검을 위해 22(SSH) 포트를 제한적으로 허용하였다. 애플리케이션은 Nginx를 통해 HTTPS 트래픽을 처리하고 내부의 Kestrel 프로세스로 전달되므로, **Kestrel 포트(5000)는 외부에 노출하지 않고 EC2 내부 통신에만 사용하도록 구성하였다.** 또한, S3 연동은 IAM Role 대신 **S3 Bucket Policy를 통해 EC2에서의 접근만 허용하는 방식으로 설정**하여 테스트 환경에서 최소 권한을 유지하면서도 빠르게 구성할 수 있도록 하였다.

| 항목 | 설정 |
| --- | --- |
| **Security Group (Inbound)** | 80 (HTTP), 443 (HTTPS), 22 (SSH) |
| **Kestrel 포트(5000)** | 내부 통신용, Security Group Inbound **미개방** |
| **EC2 → S3 업로드** | Bucket Policy로 **EC2 IP/VPC만 PutObject 허용** |
| **Browser → S3 이미지 조회** | Public Read(GetObject) 허용 |
| **Firewalld** | 기본 Disabled 또는 80/443 허용 |
| **SELinux** | Amazon Linux 2023 기본값 (Permissive) |

### 애플리케이션 디렉터리 구조

아래 표는 EC2 인스턴스 내에서 SimplCommerce 애플리케이션, Nginx 설정, systemd 서비스, Docker SQL Server가 구성되는 디렉터리 구조를 정리한 것이다.

이 구조는 **배포 경로의 일관성**, **운영 편의성**, **구성 관리의 명확성**을 위해 표준화하였다.

| 경로 | 설명 |
| --- | --- |
| `/var/www/simplcommerce` | SimplCommerce 애플리케이션 배포 디렉터리 |
| `/etc/nginx/nginx.conf` | Nginx 메인 설정 파일 |
| `/etc/nginx/conf.d/` | Reverse Proxy 및 캐시 설정 파일 저장 위치 |
| `/etc/systemd/system/simplcommerce.service` | Kestrel 기반 SimplCommerce systemd 서비스 파일 |
| `/var/log/nginx/` | Nginx Access/Error 로그 저장 경로 |
| `/var/lib/docker/` | SQL Server Docker 이미지 및 데이터 저장 위치 |
| `/home/ec2-user/deploy/` (선택) | 수동 배포 시 패키지 업로드를 위한 임시 경로(SCP) |

### 구성 목적 요약

아래 표는 본 문서에서 정의한 EC2 환경 구성이 어떤 목적을 기반으로 설계되었는지 요약한 것이다.

**초기 테스트 환경(PoC)으로서 단일 인스턴스 아키텍처를 구성**하면서도, 향후 **운영 환경으로 확장 가능한 기반을 마련**하는 데 초점을 두었다.

| 목적 | 설명 |
| --- | --- |
| **단일 인스턴스 기반 테스트 환경 구성** | Web(Nginx), App(.NET), DB(SQL Server)를 하나의 EC2 인스턴스에 통합 배치하여 손쉽게 테스트 및 검증 수행 가능 |
| **미디어 파일 분리 저장(S3 활용)** | SimplCommerce의 이미지/미디어 파일을 S3로 분리하여 EC2 디스크 부하 감소 및 파일 관리 효율성 확보 |
| **수동 배포 흐름 정립** | 초기 구축 단계에서 따라 하기 쉬운 수동 배포 절차를 기준으로 환경을 구성하고 표준화함 |
| **보안 및 접근 최소화(Least Privilege)** | S3 Bucket Policy를 사용하여 EC2에서만 S3 접근을 허용함으로써 최소 권한 기반으로 운영 |
| **향후 Multi-Tier 확장 기반 마련** | 현재는 Web/App/DB 통합 구성이나, 운영 환경에서는 서비스 계층을 분리한 Multi-Tier 아키텍처로 확장 가능 |

---

## 배포 구성 개요

본 장에서는 SimplCommerce 애플리케이션이 EC2 인스턴스에서 실행되는 방식과 Nginx, Kestrel, Docker 기반 SQL Server, Amazon S3 간의 상호작용을 포함한 전체 배포 흐름을 개요 수준에서 설명한다.

이 개요는 이후 장에서 진행할 **수동 배포 절차를 이해하기 위한 전체 맥락**을 제공한다.

### 서비스 트래픽 흐름

사용자의 요청과 애플리케이션의 이미지 업로드/조회 흐름은 다음과 같다.

1. **사용자 요청 (HTTPS)**
    
    → EC2의 Nginx가 443 포트에서 요청 수신
    
2. **Nginx Reverse Proxy**
    
    → 내부 Kestrel 프로세스(예: 5000 포트)로 요청 전달
    
3. **SimplCommerce Application**
    - DB 요청 → Docker 기반 SQL Server
    - 이미지 업로드 요청 → Amazon S3로 저장(PutObject)
4. **브라우저 이미지 조회**
    
    → S3 Public URL로 직접 접근(GetObject)
    

이 흐름을 통해 Web/App/DB/S3 간의 상호작용이 간단하고 명확하게 이루어진다.

### 수동 배포 방식 정의

본 문서에서 다루는 수동 배포 방식은 각각의 구성 요소(SQL Server, .NET Application, Nginx)를 **EC2 인스턴스에서 직접 구성하는 절차**를 기반으로 하며, 다음과 같은 흐름으로 진행된다.

1. **Docker 기반 SQL Server 구성**
    - Docker 설치
    - SQL Server 컨테이너 실행
    - SimplCommerce와 연결할 데이터베이스 준비
2. **.NET 기반 SimplCommerce 애플리케이션 배포**
    - 로컬 환경에서 `dotnet publish`
    - 배포 패키지(tar) 생성
    - EC2로 업로드(SCP)
    - `/var/www/simplcommerce` 로 배포본 교체
    - systemd 서비스 등록 및 재시작
3. **Nginx Reverse Proxy 및 HTTPS 구성**
    - Nginx 설치
    - SSL 인증서 적용
    - Reverse Proxy 설정 (443 → Kestrel 5000)
    - Nginx Cache 설정
    - Nginx 재시작 및 정상 동작 확인
4. **어플리케이션 검증 및 S3 연동 테스트**
    - SimplCommerce 페이지 정상 로딩
    - 이미지 업로드 → S3 저장 확인
    - Public URL을 통한 이미지 표시 확인
    - DB 연결 정상 확인
    - HTTPS 접속 정상 확인

수동 배포 방식은 초기 테스트 환경에서 매우 유용하다.

구성 변경 사항을 즉시 확인할 수 있고, 시스템을 구성하는 Web/App/DB 요소를 단계별로 이해할 수 있으며 자동화 이전에 전체 운영 흐름을 명확하게 파악할 수 있다.

특히 이번 문서처럼 **단일 EC2에서 모든 구성 요소를 직접 설치하는 방식**은 아키텍처 이해도 향상 및 시스템 디버깅에 큰 도움이 된다.

이제 본격적인 애플리케이션 구성을 시작한다.

---

## SQL Server (Docker) 구성

본 장에서는 EC2 인스턴스에서 **Docker 기반 SQL Server**를 구성하여 SimplCommerce 애플리케이션이 사용할 데이터베이스 환경을 준비하는 절차를 설명한다.

Docker를 활용함으로써 설치 복잡성을 줄이고, 단일 인스턴스 환경에서도 **신속하게 데이터베이스를 구성**할 수 있다.

### Docker 엔진 설치

EC2 인스턴스에서 SQL Server를 Docker로 실행하기 위해 가장 먼저 **Docker 엔진을 설치**해야 한다.

Amazon Linux 2023은 기본적으로 Docker 패키지를 제공하며, 아래 명령어로 설치 및 서비스 활성화가 가능하다.

```bash
# 시스템 패키지 최신화
sudo dnf update -y

# Docker 엔진 설치
sudo dnf install -y docker

# Docker 서비스 부팅 시 자동 실행 설정
sudo systemctl enable docker

# Docker 서비스 시작
sudo systemctl start docker

# ec2-user를 Docker 그룹에 추가 (sudo 없이 docker 명령 실행 가능)
sudo usermod -aG docker ec2-user
```

Docker 설치가 완료되면 아래 명령으로 정상 설치 여부를 확인할 수 있다.

```bash
# Docker 버전 확인
docker --version

# 실행 중인 컨테이너 목록 확인
docker ps
```

### SQL Server Docker 이미지 다운로드

다음 단계는 Microsoft 공식 SQL Server 이미지를 다운로드하는 것이다.

테스트 환경에서는 `2019-latest` 태그가 가장 널리 사용된다.

```bash
# SQL Server 2019 공식 이미지 다운로드
sudo docker pull mcr.microsoft.com/mssql/server:2019-latest

# 다운로드된 이미지 목록 확인
sudo docker images
```

### SQL Server 컨테이너 실행

이미지 다운로드가 완료되면 SQL Server 컨테이너를 실행한다.

**SA 계정 비밀번호는 SQL Server의 보안 요구사항을 충족해야 하며, 기본적으로 영문 대소문자 + 숫자 + 특수문자 조합이 필요하다.**

```bash
# SQL Server 컨테이너 실행
sudo docker run -e "ACCEPT_EULA=Y" \
  -e "SA_PASSWORD=<원하는_비밀번호>" \
  -p 1433:1433 \
  --name mssql \
  -d mcr.microsoft.com/mssql/server:2019-latest

# 실행 중인 SQL Server 컨테이너 확인
sudo docker ps

# SQL Server 초기 로그 확인
sudo docker logs simpl-sqlserver
```

### SimplCommerce 연결 문자열 설정

SQL Server가 기동되면 SimplCommerce 애플리케이션이 사용할 **데이터베이스 연결 문자열을 설정**한다.
아래의 내용을 참고한다.

- **Server**: 동일 EC2 인스턴스 내에서 접근하므로 `localhost,1433`
- **User/Password**: 컨테이너 실행 시 지정한 SA 계정 정보
- **TrustServerCertificate=True**: 테스트/개발 환경에서 인증서 없이 연결 시 필요

```json
// appsettings.json 내 설정
"ConnectionStrings": {
  "DefaultConnection": "Server=localhost,1433;Database=SimplCommerce;User Id=sa;Password=<원하는_비밀번호>;TrustServerCertificate=True;"
}
```

### SQL Server 접속 테스트 및 데이터베이스 생성

컨테이너 내부에서 SQL Server에 접속하여 정상적으로 동작하는지 확인한다.

```bash
# SQL Server CLI(sqlcmd)를 이용한 접속
sudo docker exec -it simpl-sqlserver \
  /opt/mssql-tools/bin/sqlcmd \
  -S localhost \
  -U sa \
  -P "<원하는_비밀번호>"
```

접속에 성공하면 **SimplCommerce가 사용할 빈 데이터베이스를 먼저 생성**해야 한다.

SimplCommerce는 DB 자체를 자동 생성하지 않으며, 스키마와 테이블만 자동 구성하므로 미리 데이터베이스를 만들어 두어야 한다.

아래 SQL을 순서대로 실행한다.

```sql
-- SimplCommerce 애플리케이션에서 사용할 데이터베이스 생성
CREATE DATABASE SimplCommerce;
GO

-- 생성된 데이터베이스 확인
SELECT name FROM sys.databases;
GO
```

---

## SimplCommerce 애플리케이션 구성

본 장에서는 EC2 인스턴스에서 **Kestrel 기반으로 SimplCommerce 애플리케이션을 실행**하기 위한 전체 구성 절차를 설명한다.

코드를 빌드하기 전에 필요한 설정(S3 스토리지 구성, 모듈 활성화, 프로젝트 참조 등)을 먼저 적용한 뒤, 애플리케이션을 로컬 환경에서 빌드하고 EC2 인스턴스로 배포하여 systemd 서비스로 실행하는 과정을 순차적으로 다룬다.

### 애플리케이션 사전 구성 (S3 스토리지 설정)

SimplCommerce는 기본적으로 로컬 파일 시스템을 이미지 저장소로 사용한다. 본 문서에서는 정적 파일을 Amazon S3에 저장·제공하는 구조를 사용하므로, 빌드 전에 아래 설정을 반드시 반영해야 한다.

해당 설정은 **코드 빌드(`dotnet publish`) 이전에 수행해야 하며**, S3 모듈이 포함되지 않은 상태에서 빌드하면 정상 동작하지 않는다.

#### 1. `appsettings.json` – Storage 설정 변경

SimplCommerce WebHost 프로젝트의 `appsettings.json` 파일에서 Storage Provider를 S3로 지정한다.

```json
"Storage": {
  "Provider": "S3",
  "S3": {
    "BucketName": "<your-bucket-name>",
    "Region": "ap-northeast-2",
    "BaseUrl": "https://<your-bucket-name>.s3.amazonaws.com/"
  }
}
```

본 구성에서는 EC2 → S3 접근을 **S3 Bucket Policy**로 허용하므로 AWS AccessKey/SecretKey 값은 설정하지 않는다.

#### 2. `modules.json` – S3 모듈 활성화

SimplCommerce는 모듈 기반 구조이므로, S3 Provider가 정상 동작하려면 아래의 StorageS3 모듈이 활성화되어 있어야 한다. 설정이 없다면 추가한다.

```json
{
  "Id": "SimplCommerce.Module.StorageS3",
  "IsBundledWithHost": true,
  "IsEnabled": true
}
```

- **IsEnabled = true** → S3 모듈 활성화
- **IsBundledWithHost = true** → WebHost 빌드 시 포함됨

#### 3. WebHost `.csproj` – S3 Provider 모듈 참조 추가

S3 모듈을 빌드 대상에 포함하기 위해 WebHost 프로젝트에 참조를 추가한다.

```xml
<ItemGroup>
  <ProjectReference Include="..\Modules\SimplCommerce.Module.StorageS3\SimplCommerce.Module.StorageS3.csproj" />
</ItemGroup>
```

**이 설정이 빠지면 빌드 결과물이 S3 Provider를 포함하지 않으므로 반드시 포함해야 한다.**

### 로컬 환경에서 SimplCommerce 빌드 및 패키징

사전 구성 완료 후, 로컬에서 Release 빌드 및 패키징을 수행한다.

```bash
# 로컬 환경에서 프로젝트 Root로 이동
cd <simplcommerce-root-path>

# Release 빌드
dotnet publish -c Release -o ./publish

# 빌드 디렉토리로 이동
cd publish

# tar.gz 패키지 생성
tar -czvf simplcommerce-release.tar.gz ./*

# 파일 목록 확인
ls -lh simplcommerce-release.tar.gz
```

### tar 패키지 EC2 업로드

아래 **Command Line의 괄호 안의 값을 실습 환경에 맞게 수정**하여 파일을 EC2 가상 머신에 업로드 한다.

```bash
scp -i <EC2_KEY>.pem simplcommerce-release.tar.gz ec2-user@<EC2_PUBLIC_IP>:~/deploy/
```

SSH를 사용하여 인스턴스에 접근한다.

```bash
ssh -i <EC2_KEY>.pem ec2-user@<EC2_PUBLIC_IP>
```

### EC2 배포 디렉터리 구성 및 권한 설정

```bash
# 배포 경로 준비
sudo mkdir -p /var/www/simplcommerce
sudo rm -rf /var/www/simplcommerce/*

# 패키지 압축 해제
cd /home/ec2-user/deploy
sudo tar -xzvf simplcommerce-release.tar.gz -C /var/www/simplcommerce

# 디렉터리 권한 설정
sudo chown -R ec2-user:ec2-user /var/www/simplcommerce
sudo chmod -R 755 /var/www/simplcommerce
```

### systemd 서비스 구성

```bash
# Simplcommerce.service 생성
sudo vi /etc/systemd/system/simplcommerce.service

# vim 화면에서 아래의 설정을 복사 후 저장
[Unit]
Description=SimplCommerce .NET Application
After=network.target

[Service]
WorkingDirectory=/var/www/simplcommerce
ExecStart=/usr/bin/dotnet /var/www/simplcommerce/SimplCommerce.WebHost.dll
Restart=always
RestartSec=10
User=ec2-user
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ASPNETCORE_FORWARDEDHEADERS_ENABLED=true

[Install]
WantedBy=multi-user.target

#systemd 반영 및 실행 테스트
sudo systemctl daemon-reload
sudo systemctl enable simplcommerce.service
sudo systemctl start simplcommerce.service
sudo systemctl status simplcommerce.service
```

---

## Nginx Web Server 구성

본 장에서는 Amazon EC2 인스턴스 환경에서 Nginx를 활용하여 **Reverse Proxy 및 Cache 계층을 구성하고고**, EC2 서버 자체에서 **HTTPS 종단(TLS Termination)**을 수행하는 방법을 설명드린다.

본 구성은 로드밸런서를 사용하지 않는 **단일 서버 아키텍처**를 전제로 하며, Nginx는 다음과 같은 역할을 담당한다.

- HTTPS 트래픽 처리(TLS Termination)
- Reverse Proxy 기능 수행
- Static 및 Semi-static 콘텐츠 캐싱
- 장애 시 stale cache 제공을 통한 서비스 안정성 향상

이를 통해 단일 서버 환경에서도 안정적이며 성능이 우수한 웹 서비스 운영이 가능하다.

### HTTPS 인증서 구성

단일 서버 환경에서는 Self-signed 인증서를 통해 HTTPS를 구성하실 수 있다.

생성된 인증서는 다음 경로에 저장된다.

- `/etc/nginx/ssl/simpl.crt`
- `/etc/nginx/ssl/simpl.key`

```bash
# 인증서 디렉터리 생성
sudo mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl

# Self-signed 인증서 생성
sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout simpl.key \
  -out simpl.crt
```

### Nginx Reverse Proxy 및 Cache 구성

본 절에서는 Nginx에 Cache Layer와 애플리케이션 Reverse Proxy 역할을 수행하도록 설정한다.

설정 파일 생성은 `tee` 명령어를 사용한다.

#### 1. Cache 저장소 정의

아래 명령을 통해 `proxy_cache_path` 설정을 `/etc/nginx/nginx.conf` 에 추가한다.

```bash
sudo tee -a /etc/nginx/nginx.conf > /dev/null << 'EOF'

# Cache storage definition
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=app_cache:100m \
                 max_size=10g inactive=60m use_temp_path=off;

EOF
```

#### 2. Reverse Proxy + Cache 서버 블록 생성

**아래 명령은 Reverse Proxy 및 캐싱 정책을 포함한 HTTPS(443) 서버 블록을 생성**한다.

주요 설정은 아래와 같다.

- `proxy_pass http://127.0.0.1:5000;`
    
    → Kestrel 애플리케이션으로 요청을 전달한다.
    
- `proxy_set_header X-Forwarded-Proto $scheme;`
    
    → 애플리케이션이 원 요청의 프로토콜(HTTPS)을 인지할 수 있도록 전달한다.
    
- `proxy_cache app_cache;`
    
    → 앞서 정의한 `proxy_cache_path` 설정을 참조하여 캐시를 활성화한다.
    
- `add_header X-Cache-Status $upstream_cache_status;`
    
    → 응답 헤더를 통해 캐시 HIT/MISS 여부를 확인할 수 있다
    

```bash
sudo tee /etc/nginx/conf.d/simplcommerce.conf > /dev/null << 'EOF'
server {
    listen       443 ssl;
    server_name  _;

    ssl_certificate     /etc/nginx/ssl/simpl.crt;
    ssl_certificate_key /etc/nginx/ssl/simpl.key;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:5000;

        proxy_http_version 1.1;
        proxy_set_header Host               \$host;
        proxy_set_header X-Real-IP          \$remote_addr;
        proxy_set_header X-Forwarded-For    \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto  \$scheme;

        proxy_cache            app_cache;
        proxy_cache_valid      200 302 10m;
        proxy_cache_valid      404      1m;
        proxy_cache_use_stale  error timeout updating 
                                http_500 http_502 http_503 http_504;

        add_header X-Cache-Status \$upstream_cache_status;
    }
}
EOF
```

#### 3. HTTP → HTTPS 리다이렉션 서버 블록 생성 (HTTP, 80)

모든 **HTTP(80) 요청을 HTTPS(443)로 리다이렉션**하기 위해 `/etc/nginx/conf.d/redirect.conf` 파일을 생성한다.

```bash
sudo tee /etc/nginx/conf.d/redirect.conf > /dev/null << 'EOF'
server {
    listen      80;
    server_name _;

    return 301 https://\$host\$request_uri;
}
EOF
```

파일을 분리하여 관리하는 이유는 아래와 같다.

- **역할 분리**
    1. `simplcommerce.conf` : 애플리케이션 Reverse Proxy 및 캐시, HTTPS 처리
    2. `redirect.conf` : HTTP → HTTPS 리다이렉션 전용
- **운영 편의성**
    1. 장애 대응 시 Redirect만 빠르게 비활성화 가능
    2. 설정 변경 시 영향 범위를 명확하게 구분 가능
- **자동화 및 확장성**
    - 설정 파일 단위로 기능을 ON/OFF 하기 용이하며, Ansible/SSM/Terraform 등의 자동화 도구 사용 시 템플릿 관리가 수월한다.

#### 4. Nginx 설정 검증 및 반영 및 동작 확인

설정 파일 생성 후, Nginx 설정 문법을 검증하고 서비스를 다시 로드한다.

```bash
sudo nginx -t
sudo systemctl reload nginx

# 설정 검증이 정상일 경우 아래와 유사한 메시지가 출력
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Nginx 설정 검증이 정상일 경우, Kestrel 서비스가 `active (running)` 상태인지 확인한다.

```bash
sudo systemctl status simplcommerce
```

이어서  아래 명령을 통해 응답 및 캐시 여부를 확인한다.

- 최초 요청 시: `X-Cache-Status: MISS`
- 동일 요청 반복 시: `X-Cache-Status: HIT`
    
    → 캐시가 정상적으로 동작하고 있음을 의미한다.
    

```bash
curl -I https://EC2_PUBLIC_DNS
curl -I https://EC2_PUBLIC_DNS | grep X-Cache-Status
```

---

## S3 기반 파일 스토리지 연동 구성

본 장에서는 SimplCommerce 애플리케이션의 정적 파일 및 미디어 파일을 Amazon S3로 외부화하는 방법을 설명드린다.

기존 EC2 로컬 디스크 기반 파일 저장 방식을 S3로 전환함으로써 **확장성, 안정성 및 운영 편의성**을 확보할 수 있다.

### S3 연동의 필요성

기존 구조에서는 상품 이미지, 첨부 파일 등 모든 정적 콘텐츠가 EC2의 로컬 디스크에 저장된다. **이는 다음과 같은 한계가 있다.**

- 인스턴스 교체 또는 재배포 시 파일 유실 가능성
- Auto Scaling 환경에서 인스턴스 간 파일 동기화 어려움
- 디스크 용량 관리 및 증설에 따른 운영 부담
- 정적 파일 전송까지 EC2가 담당하여 불필요한 부하 발생

**S3를 파일 스토리지로 활용할 경우 다음과 같은 이점을 얻을 수 있다.**

- 높은 내구성과 사실상 무제한에 가까운 저장 용량
- 애플리케이션 서버와 분리된 스토리지로 백업 및 마이그레이션 용이
- CloudFront와 연동 시 글로벌 캐싱 및 전송 최적화 가능
- 애플리케이션 서버를 보다 무상태(Stateless)에 가깝게 운영 가능

### S3 버킷 생성

AWS 콘솔에서 다음과 같은 설정으로 S3 버킷을 생성한다.

- **버킷 이름**:  `simplcommerce-media-dev`
- **리전**: `ap-northeast-2`
- **Object Ownership**: 기본값(AWS recommended) 사용
- **Block Public Access**: 본 문서는 **실습/데모 환경**을 가정하며, Public Read 허용을 위해 일부 옵션 해제를 전제로 한다.
- **버전 관리(Optional)**: 필요 시 활성화
- **기본 암호화**: SSE-S3 활성화 권장

### 버킷 정책 구성

본 구성에서는 S3 권한을 다음과 같이 **버킷 정책 하나로 일원화**한다.

1. 정적 파일(이미지 등)에 대한 **Public Read 허용**
2. 쓰기/삭제(업로드, 삭제)는 **EC2 인스턴스에 연결된 `EC2SessionManagerRole`** 에만 허용

EC2 인스턴스는 `EC2SessionManagerRole` Instance Profile을 사용하며, 이 역할은 기본적으로 SSM(Session Manager)용 최소 권한 역할이다.

#### 1. Public Read 허용 정책 예시 (GetObject)

정적 파일을 브라우저에서 직접 조회 가능하도록 하기 위해, 다음과 같이 `s3:GetObject` 에 대한 Public Read를 허용한다.

```json
{
  "Sid": "AllowPublicReadForObjects",
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::simplcommerce-media-dev/*"
}
```

이 정책 적용 후에는 다음 형태의 URL로 객체를 조회하실 수 있다.

- `https://simplcommerce-media-dev.s3.ap-northeast-2.amazonaws.com/<경로>/<파일명>`

운영 환경에서는 Public Read 대신, CloudFront Origin Access Control(OAC) 또는 제한된 Principal 기반 접근 제어를 사용하시는 것을 권장 드린다.

#### 2. EC2 인스턴스 프로파일에 대한 쓰기/삭제 권한 부여 (Put/Delete)

업로드(쓰기) 및 삭제는 **EC2 인스턴스 프로파일 역할인 `EC2SessionManagerRole`**에만 허용하도록 버킷 정책을 구성한다. 계정 ID는 실제 환경에 맞게 변경한다.

```json
{
  "Sid": "AllowEc2SessionManagerRoleWrite",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/EC2SessionManagerRole"
  },
  "Action": [
    "s3:PutObject",
    "s3:DeleteObject"
  ],
  "Resource": "arn:aws:s3:::simplcommerce-media-dev/*"
}
```

이렇게 하면 아래와 같이 정책이 적용된다.

- **누구나** `GetObject`(읽기)는 가능 (위 Public Read 정책 기준)
- **`EC2SessionManagerRole` 을 사용하는 EC2 인스턴스만** `PutObject`, `DeleteObject`(쓰기/삭제)를 수행 가능

즉, **IAM Role에는 S3 권한을 별도로 붙이지 않아도**, S3는 “요청을 보낸 주체(Principal)가 `EC2SessionManagerRole` 이냐?” 만 보고 버킷 정책에서 허용 여부를 판단한다.

#### 3. S3 업로드 및 조회 테스트

1. SimplCommerce 관리자 페이지에서 상품 이미지를 업로드한다.
2. 업로드 후 이미지 경로가 S3 버킷을 가리키는지 확인한다.
3. 브라우저에서 이미지 URL을 직접 열어, Public Read 정책이 정상 동작하는지 확인한다.
4. CLI로도 객체가 존재하는지 확인할 수 있다.

```bash
aws s3 ls s3://simplcommerce-media-dev
```

문제가 발생할 경우 다음 항목을 점검한다.

- 버킷 정책에 `GetObject`, `PutObject`, `DeleteObject` 가 올바르게 설정되어 있는지
- `Principal` 의 ARN이 실제 계정의 `EC2SessionManagerRole` 과 일치하는지
- SimplCommerce 설정(`BucketName`, `Region`, `BaseUrl`)이 올바른지

---

## 마무리 및 향후 확장 고려사항

본 문서에서는 SimplCommerce 애플리케이션을 Amazon EC2 단일 서버 기반에서 안정적으로 운영하기 위한 아키텍처를 구축하였다.

Nginx를 통한 HTTPS 처리 및 Reverse Proxy 구성, 정적 파일의 S3 외부화, 그리고 단일 서버 환경에 적합한 내부 DB 구성을 통해 경량 환경에서도 운영 가능한 기본 구조를 마련하였다.

본 장에서는 전체 구축 결과를 정리하고, 향후 확장 가능한 아키텍처 요소를 제시하여 문서를 마무리한다.

### 구축 완료 구성 요약

이번 구축을 통해 애플리케이션 계층, 웹 계층, 스토리지 계층 그리고 데이터베이스 계층이 다음과 같이 구성되었다.

#### 1. 애플리케이션 계층

- SimplCommerce 애플리케이션은 **Kestrel 기반**으로 동작
- `systemd` 서비스로 등록하여 자동 시작 및 상태 관리
- EC2 인스턴스 내에서 안정적으로 구동

#### 2. 웹 계층(Nginx)

- HTTPS(443)을 통한 **TLS 종단 처리**
- Reverse Proxy로 내부 애플리케이션(Kestrel)과 연동
- Proxy Cache 적용 및 `X-Cache-Status` 기반 캐시 동작 확인 가능
- 모든 HTTP 요청을 HTTPS로 리다이렉션하여 일관된 보안 정책 유지

#### 3. 스토리지 계층(Amazon S3)

- 정적 파일(상품 이미지 등)을 EC2 로컬 디스크에서 S3로 외부화
- EC2 인스턴스는 최소 권한 역할인 **EC2SessionManagerRole** 사용
- S3 접근 제어는 **IAM 정책이 아닌 S3 버킷 정책(Bucket Policy)** 에서 일원화
- 정적 파일 경로는 `BaseUrl` 을 기반으로 S3 URL 직접 호출

#### 4. 데이터베이스 계층(DB)

- 단일 서버 운영 환경의 특성상,
    
    **SimplCommerce는 EC2 인스턴스 내부 DB를 사용하도록 구성**됨
    
    (SQLite 또는 프로젝트 설정에 따른 로컬 DB 엔진)
    
- EC2 내부에 위치한 로컬 DB는
    
    소규모 서비스·실습 환경에서는 단순성과 운영 편의성이 강점
    
- 백업은 **EC2 스냅샷 기반**으로 수행 가능
- 서비스 규모 확장 시 RDS로 자연스럽게 이전 가능한 구조 유지

### 향후 확장 고려사항

단일 서버 기반 구성을 운영하다가 서비스 규모가 증가하거나 고가용성·확장성이 필요해질 경우 아키텍처를 아래 방향으로 확장할 수 있다.

#### 1. 로드밸런서(ALB) 도입

- 트래픽을 여러 EC2 인스턴스로 분산
- 헬스체크 기반의 장애 자동 감지
- ACM 기반 무료 SSL 인증서 사용 가능
- WAF와 통합하여 보안성 강화

#### 2. Auto Scaling Group 구성

- 트래픽 증가 시 자동 서버 확장
- 장애 발생 시 자동 복구
- Rolling / Blue-Green 배포와 같은 무중단 배포 구조 구성 가능

#### 3. DB 고도화 – Amazon RDS

단일 서버에 포함된 로컬 DB는 운영 부담이 적지만 규모가 커질 경우 다음과 같은 한계가 있다:

- 스토리지 확장 어려움
- 장애 발생 시 단일 장애 지점(SPOF)
- 백업/복구 자동화 제약

이를 위해 RDS 구성으로 전환하면 다음 장점이 있다.

- Multi-AZ 구성으로 고가용성 확보
- 자동 백업 및 스냅샷 관리
- 장애 조치(Failover) 기능 제공
- 애플리케이션 서버의 무상태화를 촉진

#### 4. 모니터링 및 로깅 체계 고도화

- CloudWatch Metrics 기반 서버·애플리케이션 모니터링
- CloudWatch Logs 또는 OpenSearch 기반 로그 중앙화
- Error Rate, Latency, Cache Hit Ratio 등 서비스 지표 기반 알람 구성

#### 5. CDN 기반 캐싱 확장 (CloudFront)

정적 파일의 글로벌 사용자 응답 속도를 향상 시키기 위해 CloudFront를 도입할 수 있다.

- 정적 파일: Client → CloudFront → S3
- 동적 요청: Client → CloudFront → Nginx(EC2) → Kestrel

CloudFront 도입 시 기대 효과는 다음과 같다.

- 글로벌 엣지 캐싱
- S3 및 EC2에 대한 부하 감소
- CloudFront + WAF 조합으로 보안 강화

### 마무리 하며

본 문서의 목표는 단일 EC2 인스턴스 환경에서도 안정적이고 운영 가능한 SimplCommerce 서비스 아키텍처를 구축하는 것이었다.

이번 구성에서는 다음 요소들이 완성되었다.

- HTTPS 기반의 안전한 웹 서비스 제공
- Reverse Proxy 및 Cache 구성
- 정적 파일의 S3 외부화를 통한 스토리지 구조 개선
- 단일 서버 환경에 적합한 내부 DB 구성
- S3 접근 제어를 버킷 정책으로 일원화한 클린 아키텍처

본 구성을 기반으로 하여, 향후 ALB, Auto Scaling, CloudFront, RDS 등 다양한 요소를 추가함으로써

프로덕션 수준의 고가용성, 확장성 아키텍처로 자연스럽게 확장할 수 있다.

이로써 SimplCommerce EC2 단일 서버 구축 문서를 마무리한다.

---
