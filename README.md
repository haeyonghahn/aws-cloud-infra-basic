# aws-cloud-infra-basic
## 목차
* **[EC2 Instance 접속을 위한 PuTTY 사용 방법](#EC2-Instance-접속을-위한-PuTTY-사용-방법)**
* **[서버리스 정적 웹사이트 호스팅 및 성능 가속화](#서버리스-정적-웹사이트-호스팅-및-성능-가속화)**
  * **[아키텍처 관련 기술/서비스/다이어그램/구현 순서 검토](#아키텍처-관련-기술서비스다이어그램구현-순서-검토)**
  * **[S3 Bucket 생성 및 정적 웹사이트 호스팅](#S3-Bucket-생성-및-정적-웹사이트-호스팅)**
  * **[CloudFront를 통한 웹사이트 성능 가속화](#CloudFront를-통한-웹사이트-성능-가속화)**
* **[LAMP 웹 서버 및 Application Load Balancer 구성](#LAMP-웹-서버-및-Application-Load-Balancer-구성)**
  * **[아키텍처 관련 기술/서비스/다이어그램/구현 순서 검토](#아키텍처-관련-기술서비스다이어그램구현-순서-검토-1)**
  * **[기본 네트워크 환경 구성 (VPC/Subnet/Internet Gateway/Route Table)](#기본-네트워크-환경-구성-vpcsubnetinternet-gatewayroute-table)**
  * **[Public EC2 인스턴스 생성 및 LAMP 웹서버 구성](#public-ec2-인스턴스-생성-및-lamp-웹서버-구성)**
  * **[Custom AMI를 통한 Public EC2 인스턴스 생성](#custom-ami를-통한-public-ec2-인스턴스-생성)**
  * **[EFS를 통한 네트워크 파일 시스템 구성](#efs를-통한-네트워크-파일-시스템-구성)**
  * **[Application Load Balancer를 통한 이중화 네트워크 구성 (1)](#application-load-balancer를-통한-이중화-네트워크-구성-1)**
  * **[Bastion host와 NAT Gateway를 통한 Private EC2 인스턴스의 외부 통신 구성](#bastion-host와-nat-gateway를-통한-private-ec2-인스턴스의-외부-통신-구성)**
  * **[Application Load Balancer를 통한 이중화 네트워크 구성 (2)](#application-load-balancer를-통한-이중화-네트워크-구성-2)**

## EC2 Instance 접속을 위한 PuTTY 사용 방법
Windows 환경에서 EC2에 SSH 접속을 하기 위해서 클라이언트 프로그램이 필요한데 PuTTY라는 프로그램을 사용한다.
- PuTTY 다운로드 페이지 : https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
![tempsnip](https://user-images.githubusercontent.com/31242766/208661102-462e95a7-67b5-41f2-adab-10244543043f.png)
- Mac 환경에서는 기본 터미널 프로그램 또는 iTerm과 같은 프로그램을 사용하면 된다.

### EC2 생성
- 이름 및 태그     
내 서버 이름을 지정한다. ex) demo-ec2    
![image](https://user-images.githubusercontent.com/31242766/208663753-c7e50230-6a57-4768-bc16-b7e051a5cfeb.png)
- 애플리케이션 및 OS 이미지(Amazon Machine Image)      
운영체제를 정한다.    
![image](https://user-images.githubusercontent.com/31242766/208663851-34aa66e8-47ee-4b80-952b-5681d0fff076.png)     
- 인스턴스 유형, 키 페어(로그인), 네트워크 설정       
![image](https://user-images.githubusercontent.com/31242766/208664065-734e3841-4c92-49fa-8205-6bc42f3c92ba.png)
- 스토리지 구성    
![image](https://user-images.githubusercontent.com/31242766/208664162-ed3ae3a4-a1ca-4afb-9696-40fa0fb67543.png)

### PuTTY Key .ppk 파일 생성
- load 클릭   
![image](https://user-images.githubusercontent.com/31242766/208664378-f733e6c8-96ed-437e-ab54-af6de840b9cd.png)
- Save private key    
![image](https://user-images.githubusercontent.com/31242766/208664635-b681e507-11b9-415b-ba7e-628d03e8480a.png)

### PuTTY로 EC2 접속
- `Session` -> `Host Name`에 EC2 퍼블릭 IP 입력        
![image](https://user-images.githubusercontent.com/31242766/208665058-726a7c29-4419-469b-b632-12f98794b9c3.png)    
- `Connection` -> `Auth` -> `Credentials` 생성한 .ppk 파일 입력    
![image](https://user-images.githubusercontent.com/31242766/208665640-ea8e5c1d-f3ed-4b21-bfde-7298bb8c2570.png)
- EC2 접속    
![image](https://user-images.githubusercontent.com/31242766/208665930-eaf66202-2565-4fff-b5a5-8d808e7ac791.png)

## 서버리스 정적 웹사이트 호스팅 및 성능 가속화
서버가 없어도 구성이 가능한 정적 웹 사이트를 호스팅하고 콘텐츠 전송 네트워크(CDN) 서비스를 이용하여 웹 사이트 성능을 향상시킬 수 있다.

### 아키텍처 관련 기술/서비스/다이어그램/구현 순서 검토
기본적으로 대부분 서울 지역으로 진행하지만 웹 페이지 로딩 속도를 테스트하고 비교하기 위해 서울 지역과 멀리 떨어진 곳으로 설정하여 진행해 본다. `S3 버킷`에 저장되어 있는 콘텐츠를 통해서 웹 포스팅이 가능하다. 콘텐츠가 저장되어 있는 지역이 멀리 떨어져 있거나 콘텐츠의 크기가 크면 로딩이 지연되는 현상이 발생한다. 이러한 문제를 해결해주기 위한 서비스가 콘텐츠 전송 네트워크(CDN)인 `CloudFront` 이다. S3 버킷에 저장되어 있는 콘텐츠들이 캐싱되어 빠른 속도로 사용자들에게 전송될 수 있다.
![image](https://user-images.githubusercontent.com/31242766/208666848-65570499-752b-4700-b79a-5e4dd0dbfc0d.png)

### S3 Bucket 생성 및 정적 웹사이트 호스팅
1. S3 Bucket 생성
   - 일반 구성    
   ![image](https://user-images.githubusercontent.com/31242766/208670038-0de5b866-a2cf-44f2-b0ce-dd2e7ea81095.png)
   - 객체 소유권, 이 버킷의 퍼블릭 액세스 차단 설정, 버킷 버전 관리
   ![image](https://user-images.githubusercontent.com/31242766/208670254-1d73fd50-daae-4317-afaa-39d5df9a3792.png)
   - 태그, 기본 암호화   
   ![image](https://user-images.githubusercontent.com/31242766/208670329-a725ad9d-5744-4acd-8726-3166134f182a.png)

3. Object(File) 업로드
   - mycar.html
   - car.jpg   
   ![image](https://user-images.githubusercontent.com/31242766/208670969-87302a15-03b5-4684-8e69-6ef598e91c55.png)
4. 정적 웹 사이트 호스팅 기능 활성화
   - Amazon S3 -> 버킷 -> lab-s3-web-hosting-by-haeyong -> 정적 웹 사이트 호스팅 설정
   ![image](https://user-images.githubusercontent.com/31242766/208671705-415a484e-3c9e-478a-84d4-09f48a883bbe.png)
6. Bucket과 Object에 대한 액세스 정책 설정
   - Amazon S3 -> 버킷 -> lab-s3-web-hosting-by-haeyong -> 퍼블릭 액세스 차단 편집(버킷 설정)
   ![image](https://user-images.githubusercontent.com/31242766/208672859-f0ab481a-0a54-4d60-9d26-8cbc8186a93d.png)
   - Amazon S3 -> 버킷 -> lab-s3-web-hosting-by-haeyong -> 버킷 정책 편집 -> 새 문 추가   
   ![image](https://user-images.githubusercontent.com/31242766/208679819-6a9880eb-a5eb-4e5c-982d-48159e9d9ca6.png)    
   - Amazon S3 -> 버킷 -> lab-s3-web-hosting-by-haeyong -> 버킷 정책 편집 -> 정책 생성기
     - Select Type of Policy : S3 Bucket Policy
     - Effect(접근하는 사람을 선택할 것인지 안선택할 것인지) : Allow
       - Allow : 모두 허용
       - Deny : 선택해서 받음
     - Principal(접근할 수 있는 사람) : *
       - "*" : 모두
       - 접근할 수 있는 사람 명시 : 그 사람만
     - Actions : GetObject
     - Amazon Resource Name (ARN) (접근하고자 하는 S3의 버킷 ARN/경로) : arn:aws:s3:::lab-s3-web-hosting-by-haeyong/*
     ![image](https://user-images.githubusercontent.com/31242766/208694873-36e5c2f5-5a94-44b1-9789-e66e7d5bb6df.png)
     ![image](https://user-images.githubusercontent.com/31242766/208697131-d4b07b88-556d-42f5-8a30-97ad4feb7b51.png)
     ![image](https://user-images.githubusercontent.com/31242766/208697333-7a4b0c37-da69-4939-835e-f61db542fdb5.png)
7. 웹 브라우저에서 웹 사이트 작동 확인
![image](https://user-images.githubusercontent.com/31242766/208697648-7eef353b-6ffa-4f28-b890-d260333b7fd7.png)
![image](https://user-images.githubusercontent.com/31242766/208697709-1700858a-d24d-4c53-acb6-e1ba482ea0af.png)
### CloudFront를 통한 웹사이트 성능 가속화
1. CloudFront Distribution 생성 (Origin, Cache behavior 등 설정)
   - 원본
   ![image](https://user-images.githubusercontent.com/31242766/209772674-ec9f08a4-fd2e-4ce7-8b7c-7e9c38aaea87.png)
   ![image](https://user-images.githubusercontent.com/31242766/209772738-84c2c783-71ab-49d6-8e34-7dfa407fa316.png)
   - 기본 캐시 동작
   ![image](https://user-images.githubusercontent.com/31242766/209772915-24f37fce-dabd-4e41-9253-fe17f2338323.png)   
     - Compress objects automatically (자동으로 객체 압축)
       - origin에서 가져온 리소스롤 성능이나 속도를 높이기 위해 자동으로 압축하는 기능이다.
   - 캐시 키 및 원본 요청   
   캐시 키는 캐싱된 객체가 가지고 있는 고유 식별자. 키 값이다. 사용자의 요청에 의한 캐시 키와 이전 요청으로 생성되었던 캐시 키가 동일할 경우 `캐시 hit`라고 표현한다.
   `캐시 hit`가 발생할 경우 해당 `Object`, `콘텐츠`가 CloudFront `edge location` 에서 최종 사용자에게 전달이 되기 때문에 `origin 서버`의 부하를 감소시킬 수 있고 
   최종 사용자의 delay time을 줄일 수 있게 된다. 
   ![image](https://user-images.githubusercontent.com/31242766/209773536-a22a1f38-b4d9-4832-9322-8e15c2a91bf6.png)
   - 함수 연결 - 선택 사항   
   ![image](https://user-images.githubusercontent.com/31242766/209775128-f67f3243-e1d9-4533-867e-d4083d5fcec4.png)
   - 설정   
   ![image](https://user-images.githubusercontent.com/31242766/209775582-58c27dda-ad1c-4746-a97d-c85802c874f8.png)
2. 웹 브라우저에서 CloudFront Distribution 작동 확인
3. 웹 사이트 성능 테스트
   - S3 정적 웹 사이트 호스팅을 통한 콘텐츠 로드 속도
   - CloudFront를 통한 콘텐츠 로드 속도

## LAMP 웹 서버 및 Application Load Balancer 구성
### 아키텍처 관련 기술/서비스/다이어그램/구현 순서 검토
__개요__   
클라우드 네트워크 환경에 `L`inux 가상 서버에 `A`pache 웹 서버, `M`ySQL 데이터베이스, `P`HP 어플리케이션을 구성하고 Application Load Balancer를 이용하여 
이중화 된 네트워크를 구성한다.

__사용하는 AWS 서비스__   
- Amazon VPC(VPC, Subnet, Internet Gateway, Route Table, NAT Gateway 등)
- Amazon EC2
- Amazon EBS
- Amazon EFS
- Elastic Load Balancer - Application Load Balancer

![image](https://user-images.githubusercontent.com/31242766/209809858-b909c846-3ab1-493a-8094-b86ae39f23df.png)

### 기본 네트워크 환경 구성 (VPC/Subnet/Internet Gateway/Route Table)
![image](https://user-images.githubusercontent.com/31242766/209810091-73af8f78-8b64-46e3-b742-0af10dbf1dee.png)
1. VPC 생성   
  - VPC -> VPC -> VPC 생성   
  ![image](https://user-images.githubusercontent.com/31242766/209882149-31fe2e2a-1ff6-4433-a499-bc78b965e121.png)     
  - VPC -> VPC -> VPC ID -> VPC 설정 편집 (DNS 호스트 이름 활성화 체크)
  ![image](https://user-images.githubusercontent.com/31242766/209887865-775ac8d6-0268-4526-8907-621181d23fac.png)   
    - DNS 호스트 이름 활성화 : VPC 내부에서 생성되는 인스턴스에 퍼블릭 DNS 호스트 이름을 할당해주는 기능이다.
> VPC 생성과 함께 만들어지는 리소스들
> 1. 라우팅 테이블   
> 라우팅 테이블은 실제로 서브넷과 연결된다. VPC 에도 연결되어 있는데 여기에 연결된 라우팅 테이블이 VPC 에 속한 서브넷을 생성할 때 서브넷과 자동으로 연결된다.
> 해당 대역의 IP가 유입되게 되면 VPC 내에서 리소스를 검색한다.   
> 2. DHCP 옵션 셋   
> 도메인 네임 서버, NTP 서버, NetBIOS 서버 등의 정보를 가지고 있으며 보통은 기본값을 사용한다.   
> 3. ACL과 시큐리티 그룹   
> ACL은 서브넷의 앞단에서 방화벽 역할을 하는 리소스이다. ACL에는 인바운드와 아웃 바운드 규칙이 따로 있다. VPC와 함께 생성되는 ACL에는 
> 인바운드와 아웃바운드 규칙 각각 2개의 규칙을 가지고 있다. 규칙의 번호는 우선순위를 나타낸다. 번호가 * 인 규칙은 다른 어떤 룰에도 매치하지 않을 경우 
> 사용하는 기본 규칙이다. 기본 규칙은 모든 트래픽을 차단(Deny) 한다. 하지만 이 ACL에서는 기본 규칙이 사용되지 않는다. 왜냐하면 100번 규칙에서 모든 
> 트래픽을 허용(Allow)하고 있기 때문이다.
> ![image](https://user-images.githubusercontent.com/31242766/209889745-aa55702c-2ba6-413c-b09a-5a2a0f9e2ab2.png)

2. Subnet 생성
  - VPC -> 서브넷 -> 서브넷 생성   
  Subnet은 VPC 의 CIDR 블럭. VPC 의 IP 대역을 나누는 것이기 때문에 VPC 의 하위 네트워크인 서브넷을 만들 때, 서브넷의 CIDR 블럭이 VPC CIDR 블럭보다 작아야한다.   
  ![image](https://user-images.githubusercontent.com/31242766/209890591-54cc7a8f-642a-46cb-8690-a682e266cbff.png)
  ![image](https://user-images.githubusercontent.com/31242766/209890713-1f950690-00ff-402e-830b-d8e9a12413c9.png)
  ![image](https://user-images.githubusercontent.com/31242766/209890749-eb090871-2340-4e45-b660-d72686126159.png)
  ![image](https://user-images.githubusercontent.com/31242766/209890793-9b4aec24-b060-4a05-9344-5425831f3a78.png)
  ![image](https://user-images.githubusercontent.com/31242766/209890969-f7d8d692-1e38-4b5d-880a-1f818827184d.png)   
  동일하게 `private-subnet-c2` 도 생성한다.   
  Subnet을 생성했으므로 EC2와 같은 리소스들이 생성될 수 있는 위치가 만들어졌다. 하지만 리소스들 사이, 외부 인터넷 사이에 트래픽이 이동할 수 있는 경로를 설정해주지 않았다.
  이런 역할을 하는 네트워크 요소가 Internet Gateway와 Route Table이다.
  
3. Internet Gateway 생성
  - VPC -> 인터넷 게이트웨이 -> 인터넷 게이트웨이 생성   
  ![image](https://user-images.githubusercontent.com/31242766/209891341-71efbf6b-b452-4dc5-a186-6212eaf8f3f1.png)
  - VPC -> 인터넷 게이트웨이 -> VPC에 연결   
  ![image](https://user-images.githubusercontent.com/31242766/209891401-0e2b106d-f752-404e-93c6-9c9825aff574.png)
4. Route Table 생성 및 Route 설정   
네트워크 통신 함에 있어 목적지, 대상지 경로에 데이터 패킷 이동 정보를 구성하는 규칙이다. Subnet에 위치한 EC2 리소스와 같이 것들이 목적지, 대상지에 따라서 어떤 경로를 통해 
트래픽이 이동할 것인지에 대한 규칙을 설정해놓은 테이블이다.   
- VPC -> 라우팅 테이블 -> 라우팅 테이블 생성
![image](https://user-images.githubusercontent.com/31242766/209891779-c4298049-2ec8-49cd-b805-bf9da798ec48.png)   
- VPC -> 라우팅 테이블 -> 라우팅 테이블 ID -> 서브넷 연결 편집   
![image](https://user-images.githubusercontent.com/31242766/209892096-2b7c0474-1a85-4ac4-9b6a-5368bd5ff2fc.png)   
Internet gateway를 VPC에 붙인 것처럼 라우팅 테이블이 어떤 서브넷에 대한 트랙픽 경로인지에 정해주어야한다. `public-subnet-a1`, `public-subnet-c1` 서브넷에 연결하도록 한다.   
- VPC -> 라우팅 테이블 -> 라우팅 테이블 ID -> 라우팅 편집    
public subnet에 위치한 EC2 인스턴스와 같은 리소스들이 외부 인터넷과 통신할 수 있도록 라우팅 테이블에 인터넷 게이트웨이를 경로로 추가한다.   
![image](https://user-images.githubusercontent.com/31242766/209892961-84fb9aa8-a807-4f08-a9f6-7f7b967e197e.png)   
> 참고 : Destination이 `10.1.0.0/16` 이고 Target `local`로 되어 있는 것은 
> 기본적으로 내 VPC CIDR 블럭 내에서는 트래픽이 이동이 가능하다는 것을 의미한다.

- private subnet에 대한 라우팅 테이블 생성    
![image](https://user-images.githubusercontent.com/31242766/209896519-0f4f3843-ed73-490a-89ab-803dba1771ef.png)
![image](https://user-images.githubusercontent.com/31242766/209896867-e08b6e3d-7a26-4258-b2a3-3caae3c099e7.png)   
`private-subnet-a1` 과 마찬가지로 `private-subnet-c1`, `private-subnet-a2`, `private-subnet-c2` 에 대한 라우팅 테이블을 생성한다.

### Public EC2 인스턴스 생성 및 LAMP 웹서버 구성
![image](https://user-images.githubusercontent.com/31242766/209810256-dd536659-cb46-4c38-9931-c436914facf2.png)
1. EC2 생성   
- 이름 및 태그, 애플리케이션 및 OS 이미지(Amazon Machine Image)   
![image](https://user-images.githubusercontent.com/31242766/209909357-157e1eb3-1774-42c5-b3f3-38ec365baf02.png)
- 키 페어(로그인)   
![image](https://user-images.githubusercontent.com/31242766/209909962-ed35128b-4129-4881-a2f9-beb6993d9beb.png)
- 네트워크 설정   
![image](https://user-images.githubusercontent.com/31242766/209910983-78feee16-bc5d-434d-a48a-159e09003676.png)   
![image](https://user-images.githubusercontent.com/31242766/209911018-ea1c0316-5d2a-4122-9026-8d15158c418f.png)
- Storage(EBS) 
![image](https://user-images.githubusercontent.com/31242766/209911197-f84060c4-a047-4174-a158-9443ed2ed0a6.png)
- 파일시스템
![image](https://user-images.githubusercontent.com/31242766/209911612-9194c8a4-6a67-4a59-b53b-341f5c26c6ee.png)
- 고급 세부 정보
![image](https://user-images.githubusercontent.com/31242766/209913372-8c4824e6-3f2f-49ea-bc36-4398c435f54e.png)
![image](https://user-images.githubusercontent.com/31242766/209913421-246f6943-9d88-425f-a2bd-bb9f6a19cd4b.png)
![image](https://user-images.githubusercontent.com/31242766/209913526-639d9b77-a93c-4997-afa9-4fcbcc049176.png)

2. Elastic IP 할당
- EC2 -> 탄력적 IP 주소 -> 탄력적 IP 주소 할당    
탄력적 IP 주소를 생성한다.   
![image](https://user-images.githubusercontent.com/31242766/209913943-1df900ad-4cda-411f-8852-88faa70c30af.png)    
- EC2 -> 탄력적 IP 주소 -> 3.37.180.76 -> 탄력적 IP 주소 연결     
탄력적 IP 주소와 EC2 인스턴스를 연결한다.      
![image](https://user-images.githubusercontent.com/31242766/209914187-7157be85-c04f-4a37-9afc-33ec7acda037.png)
![tempsnip](https://user-images.githubusercontent.com/31242766/209915947-27d15264-d31e-4b2f-a908-13846af1a10f.png)

3. Index.php 파일 생성
- EC2 인스턴스에 접속   
![image](https://user-images.githubusercontent.com/31242766/209918540-b1baabf0-3e98-4c5b-b25d-4ac83c02f746.png)   
- `root` 로 이동    
![image](https://user-images.githubusercontent.com/31242766/209918628-89be94fb-9062-4f06-8255-ffbe7e1a94a1.png)
- `/var/www/html` 경로에 `index.php` 파일 생성
![image](https://user-images.githubusercontent.com/31242766/209918825-b0ccb64f-391d-488e-941f-57c599e88f9e.png)   
```php
<?php include "dbinfo.inc"; ?>
<html>
<body>
<h1>Instance data</h1>
<?php

  echo "<table>";
  echo "<tr><th>Data</th><th>Value</th></tr>";

  $urlRoot="http://169.254.169.254/latest/meta-data/";

  echo "<tr><td>Instance ID</td><td><i>" . file_get_contents($urlRoot . 'instance-id') . "</i></td><tr>";
  echo "<tr><td>Private IP Address</td><td><i>" . file_get_contents($urlRoot . 'local-ipv4') . "</i></td><tr>";
  echo "<tr><td>Public IP Address</td><td><i>" . file_get_contents($urlRoot . 'public-ipv4') . "</i></td><tr>";
  echo "<tr><td>Availability Zone</td><td><i>" . file_get_contents($urlRoot . 'placement/availability-zone') . "</i></td><tr>";

  echo "</table>";

?>
<h1>RDS Practice</h1>
<?php

  /* Connect to MySQL and select the database. */
  $connection = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD);

  if (mysqli_connect_errno()) echo "Failed to connect to MySQL: " . mysqli_connect_error();

  $database = mysqli_select_db($connection, DB_DATABASE);

?>

<!-- Display table data. -->
<table border="1" cellpadding="2" cellspacing="2">
  <tr>
    <td>ID</td>
    <td>NAME</td>
    <td>ADDRESS</td>
  </tr>

<?php

$result = mysqli_query($connection, "SELECT * FROM SAMPLE");

while($query_data = mysqli_fetch_row($result)) {
  echo "<tr>";
  echo "<td>",$query_data[0], "</td>",
       "<td>",$query_data[1], "</td>",
       "<td>",$query_data[2], "</td>";
  echo "</tr>";
}
?>

</table>

<!-- Clean up. -->
<?php

  mysqli_free_result($result);
  mysqli_close($connection);

?>

</body>
</html>
```

4. 웹 브라우저에서 LAMP 웹 서버 작동 테스트    
EC2 인스턴스 `퍼블릭 IP 주소`로 접속    
![image](https://user-images.githubusercontent.com/31242766/209919324-066e34ff-afff-4b5f-a97c-1d69dcffce0e.png)

### Custom AMI를 통한 Public EC2 인스턴스 생성
![image](https://user-images.githubusercontent.com/31242766/209810557-84d0b07f-28a2-40cd-84ec-d3d417f582dc.png)
1. Custom AMI 생성
2. Custom AMI를 통해 EC2 추가 생성
3. 웹 브라우저에서 LAMP 웹 서버 작동 테스트

### EFS를 통한 네트워크 파일 시스템 구성
![image](https://user-images.githubusercontent.com/31242766/209810645-9e6827b4-e033-4661-b663-3250263e043a.png)
1. EFS용 Security Group 생성
2. EFS 생성
- Availability
- Lifecycle
- Performance/Throughput mode 
- Network 등
3. EFS-EC2 마운트
4. 웹 브라우저를 통한 EFS 마운트 테스트

### Application Load Balancer를 통한 이중화 네트워크 구성 (1)
![image](https://user-images.githubusercontent.com/31242766/209810858-7ebf2ddd-b2aa-48c4-82ab-5cf893af2e4c.png)
1. Target group 생성
- Target type
- Protocol/Port
2. Application Load Balancer 구성
- Scheme
- Network
- Security Group
- Listener/Rule 등
3. 웹 브라우저를 통한 Application Load Balancer 작동 테스트

### Bastion host와 NAT Gateway를 통한 Private EC2 인스턴스의 외부 통신 구성
![image](https://user-images.githubusercontent.com/31242766/209811046-a1fdafc9-b231-43f8-901b-3b9e5f7b9d0f.png)
1. Private subnet에 EC2 생성
2. Public subnet의 EC2를 통해 Private subnet의 EC2에 접속 (+ key pair 생성)
3. NAT Gateway 생성
4. Route table 설정
5. Private subnet의 EC2의 외부 통신 테스트

### Application Load Balancer를 통한 이중화 네트워크 구성 (2)
![image](https://user-images.githubusercontent.com/31242766/209811216-c4d830c7-7ce8-432a-a709-617f74d87ef3.png)
1. Target group 생성
2. Application Load Balancer 구성
3. 웹 브라우저를 통한 Application Load Balancer 작동 테스트
