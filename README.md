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
AWS EC2 서비스는 말 그대로 `하나의 서버를 제공하여 편하게 관리할 수 있도록 도와주는 서비스이다.` AWS EC2의 가장 큰 장점 중 하나는, 하나의 서버에 다양한 프로그램이 설치 되어있는 형태 그대로를 이미지(AMI)로 만들 수 있다는 점이다. 그렇기 때문에 컨테이너 기술을 잘 모르는 사람들도 편하게 서버를 이미지화하여 사용할 수 있다.   
서버를 이미지화 했을 때 얻을 수 있는 가장 큰 이점으로는 서버가 정상적으로 동작했을 때의 상태를 미리 이미지로 저장해 놓을 수 있다는 것이다. 그래서 서버에 문제가 생겼을 때는 현재의 서버를 내려 버리고, 이미지(AMI)로 만들어 놓았던 것을 그대로 AWS EC2 인스턴스로 만들 수 있다.

- EC2 -> 인스턴스 -> EC2 인스턴스 ID -> 이미지 생성   
![image](https://user-images.githubusercontent.com/31242766/209920663-e1d9672a-8316-4ce5-b54e-7ca0bf7515f2.png)

2. Custom AMI를 통해 EC2 추가 생성
- 이름 및 태그, 애플리케이션 및 OS 이미지(Amazon Machine Image)   
![image](https://user-images.githubusercontent.com/31242766/209921033-a6f7d263-0481-4411-ab24-70db5fc95112.png)   
- 키 페어(로그인)   
![image](https://user-images.githubusercontent.com/31242766/209921153-c5e57ac5-8633-41fa-8f59-9e0c10a2b7e4.png)    
- 네트워크 설정   
![image](https://user-images.githubusercontent.com/31242766/209922056-ae93af6f-3379-4633-9ec6-87f4533ae444.png)   
- 스토리지 구성   
![image](https://user-images.githubusercontent.com/31242766/209922110-13d0df04-4536-474f-a16d-bb26661a6675.png)   
- 고급 세부 정보   
> 중요 : 고급 세부 정보 설정은 `Public EC2 인스턴스 생성 및 LAMP 웹서버 구성`에서 EC2 생성 시 설정한 고급 세부 설정을 따른다. 하지만 
> `고급 세부 정보 -> 사용자 데이터`는 `public-ec2-a1`을 만들 때 인스턴스가 생성되는 과정에서 LAMP 웹 서버 구성에 필요한 주 패키지를 설치하게 했지만
> 이번 인스턴스는 해당 패키지들이 이미 설치되어 있는 인스턴스의 이미지로 만든 `AMI`를 사용하기 때문에 `사용자 데이터`는 별도로 추가하지 않고 
> 진행한다.

- Elastic IP 할당   
`Public EC2 인스턴스 생성 및 LAMP 웹서버 구성`에서 탄력적 IP 주소를 설정 정보와 동일하게 진행한다.   
![image](https://user-images.githubusercontent.com/31242766/209923480-ddf8308b-e6ff-4d11-8e31-61686f87dd04.png)   
![image](https://user-images.githubusercontent.com/31242766/209923651-998ce7e2-208e-4b96-a498-e037f41d3960.png)
![image](https://user-images.githubusercontent.com/31242766/209923788-2d2b7afa-e559-47c9-9704-2635e1a36dfb.png)

3. 웹 브라우저에서 LAMP 웹 서버 작동 테스트
![image](https://user-images.githubusercontent.com/31242766/209923946-90959f8f-827c-4680-85be-3a9d58cc4c6e.png)

### EFS를 통한 네트워크 파일 시스템 구성
![image](https://user-images.githubusercontent.com/31242766/209810645-9e6827b4-e033-4661-b663-3250263e043a.png)
1. EFS용 Security Group 생성    
- VPC -> 보안 그룹 -> 보안 그룹 생성   
![image](https://user-images.githubusercontent.com/31242766/209927768-3271a2fb-551e-4baf-a573-40bfeb1d4e20.png)   
EFS 에 대한 Security Group 을 구성할 때 EFS와 연결하고자 하는 인스턴스 Security Group 을 선택한다. public-ec2-sg 에서 나오는 NFS 트래픽을 마운트하는 것이다. 

2. EFS 생성
- Amazon EFS -> 파일 시스템 -> 생성
  - 일반   
  ![image](https://user-images.githubusercontent.com/31242766/210023356-b866d726-8742-4983-9e50-a80db6214fd2.png)   
  - 성능 설정   
  ![image](https://user-images.githubusercontent.com/31242766/210023645-ba337c28-9789-4d54-9d9c-86fd8ce14967.png)   
  - 네트워크   
  ![image](https://user-images.githubusercontent.com/31242766/210023817-6a2af3a4-d83a-40af-9cf2-5e5cf39da0a0.png)   
  - 파일 시스템 정책   
  ![image](https://user-images.githubusercontent.com/31242766/210023864-f80212a9-4f0b-4992-a08f-5710a1acd05f.png)   

3. EFS-EC2 마운트
- EC2 인스턴스에 접속   
![image](https://user-images.githubusercontent.com/31242766/210024195-f3af9bc7-81c4-4c26-a773-eed52c7127f5.png)   
`df -h` 명령어는 파일 시스템의 디스크 공간을 확인하는 명령어이다.   
현재는 마운트된 EFS가 없는 것을 확인할 수 있다.   
- EFS 마운트 헬퍼 설치   
```linux
[ec2-user@ip-10-1-1-143 ~]$ sudo su
[root@ip-10-1-1-143 ec2-user]# yum install amazon-efs-utils -y
```
- EFS 마운트 포인트 생성    
마운트 포인트는 사실 어디에 만들어도 상관은 없다. 하지만 실습을 위해서 아래 경로에 생성을 하도록 한다.   
```linux
[root@ip-10-1-1-143 ec2-user]# cd /var/www/html
[root@ip-10-1-1-143 html]# ls -la
total 4
drwxrwsr-x 2 ec2-user apache   23 Dec 29 07:31 .
drwxrwsr-x 4 ec2-user apache   33 Dec 29 06:45 ..
-rw-r--r-- 1 root     apache 1524 Dec 29 07:31 index.php
[root@ip-10-1-1-143 html]# mkdir efs
[root@ip-10-1-1-143 html]# ls -la
total 4
drwxrwsr-x 3 ec2-user apache   34 Dec 30 00:54 .
drwxrwsr-x 4 ec2-user apache   33 Dec 29 06:45 ..
drwxr-sr-x 2 root     apache    6 Dec 30 00:54 efs
-rw-r--r-- 1 root     apache 1524 Dec 29 07:31 index.php
```
- EFS-EC2 연결
![tempsnip](https://user-images.githubusercontent.com/31242766/210024580-652f4764-c589-492c-9e51-05b1e32eb05f.png)   
```linux
[root@ip-10-1-1-143 html]# sudo mount -t efs -o tls fs-08b2f0b43ab1b71a7:/ efs
[root@ip-10-1-1-143 html]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        474M     0  474M   0% /dev
tmpfs           483M     0  483M   0% /dev/shm
tmpfs           483M  476K  482M   1% /run
tmpfs           483M     0  483M   0% /sys/fs/cgroup
/dev/xvda1      8.0G  1.9G  6.2G  23% /
tmpfs            97M     0   97M   0% /run/user/1000
127.0.0.1:/     8.0E     0  8.0E   0% /var/www/html/efs
[root@ip-10-1-1-143 html]#
```

4. 웹 브라우저를 통한 EFS 마운트 테스트
- S3 버킷에 있는 `mycar.jpg` 객체 URL 복사하여 EFS 에 다운로드   
![tempsnip](https://user-images.githubusercontent.com/31242766/210024876-b6ae8379-804c-4096-a6aa-b174b4e65016.png)   
```linux
[root@ip-10-1-1-143 efs]# wget https://lab-s3-web-hosting-by-haeyong.s3.ap-northeast-2.amazonaws.com/car.jpg
[root@ip-10-1-1-143 efs]# ls -la
total 8508
drwxr-xr-x 2 root     root      6144 Dec 30 01:06 .
drwxrwsr-x 3 ec2-user apache      34 Dec 30 00:54 ..
-rw-r--r-- 1 root     root   8705340 Dec 20 12:47 car.jpg
```
- `mycar.html` 객체 URL 도 동일하게 다운로드 한다.
```linux
[root@ip-10-1-1-143 efs]# wget https://lab-s3-web-hosting-by-haeyong.s3.ap-northeast-2.amazonaws.com/mycar.html
[root@ip-10-1-1-143 efs]# ls -la
total 8512
drwxr-xr-x 2 root     root      6144 Dec 30 01:07 .
drwxrwsr-x 3 ec2-user apache      34 Dec 30 00:54 ..
-rw-r--r-- 1 root     root   8705340 Dec 20 12:47 car.jpg
-rw-r--r-- 1 root     root        94 Dec 20 12:47 mycar.html
[root@ip-10-1-1-143 efs]#
```
- `http://EC2인스턴스퍼블릭IP/efs/mycar.html`로 접속
![image](https://user-images.githubusercontent.com/31242766/210025194-6ad901e3-d2a1-458e-9a3c-0ccc9842cb85.png)

- `public-ec2-c1`도 동일하게 진행한다. 
`public-ec2-c1` EFS 경로로 들어가보면 `public-ec2-a1` 에서 올렸던 파일들을 확인할 수 있다.
```linux
[root@ip-10-1-2-34 html]# cd /var/www/html/efs
[root@ip-10-1-2-34 efs]# ls -la
total 8512
drwxr-xr-x 2 root     root      6144 Dec 30 01:07 .
drwxrwsr-x 3 ec2-user apache      34 Dec 30 01:20 ..
-rw-r--r-- 1 root     root   8705340 Dec 20 12:47 car.jpg
-rw-r--r-- 1 root     root        94 Dec 20 12:47 mycar.html
[root@ip-10-1-2-34 efs]#
```
![image](https://user-images.githubusercontent.com/31242766/210025578-ff32813b-c995-4753-aee4-29115e3743fd.png)

### Application Load Balancer를 통한 이중화 네트워크 구성 (1)
Application Load Balancer를 통해서 이중화 네트워크를 구성하고 트래픽을 분산시킨다.   
![image](https://user-images.githubusercontent.com/31242766/209810858-7ebf2ddd-b2aa-48c4-82ab-5cf893af2e4c.png)   
1. Target group 생성   
- 그룹 세부 정보 지정   
로드 밸런서는 기본적으로 트래픽을 받으면 리스너를 통해 타겟 그룹으로 전달하게 된다.   
![image](https://user-images.githubusercontent.com/31242766/210025961-f606b01b-12f4-4166-bebb-ebe09dec0297.png)
![image](https://user-images.githubusercontent.com/31242766/210026064-a8b1434c-b6b2-4642-a846-1e11893349df.png)   
Protocol과 Port는 Application Load Balancer와 타겟이 되는 EC2 인스턴스 사이의 통신에 대한 것이다. 그래서 EC2 인스턴스는
여기서 설정하는 Protocol과 Port 요청만 받아들이겠다는 의미가 된다.   
![image](https://user-images.githubusercontent.com/31242766/210026288-0f22681f-8799-4632-ac6b-d934adacb97f.png)
![image](https://user-images.githubusercontent.com/31242766/210026350-61a9e60e-1e2c-42ee-9d2b-c18c2c00d41e.png)   
- 대상 등록   
![tempsnip](https://user-images.githubusercontent.com/31242766/210026462-6faa34ac-ed26-4923-97fb-59c193c68559.png)

2. Application Load Balancer 구성
- 기본구성   
![image](https://user-images.githubusercontent.com/31242766/210026611-f5abae71-1009-4dbd-8092-a6d1ef38a532.png)   
- Network mapping     
![image](https://user-images.githubusercontent.com/31242766/210026796-42d01ad5-7784-4f25-a309-a1c794b00ac8.png)   
- 보안 그룹    
새 보안 그룹을 생성하여 등록한다.   
![image](https://user-images.githubusercontent.com/31242766/210026980-c04d4fbf-38f0-4118-950e-3e92481b4386.png)    
![image](https://user-images.githubusercontent.com/31242766/210027028-f3fd19b7-e0f8-4062-a2fe-9c23e8b961ad.png)    
![image](https://user-images.githubusercontent.com/31242766/210027077-f8fc08d1-317b-4a7f-9ede-3a73eab48b56.png)    
- 리스너 및 라우팅    
리스너가 로드밸런서로 들어온 트래픽, 요청을 받아들이고 정보를 분석해서 어느 타겟 그룹을 트래픽을 전달할지 수행한다.    
이 단계에서는 외부 인터넷과 Application Load Balancer 사이의 통신에 대한 프로토콜과 포트를 지정한다.   
기본 작업은 리스너가 전달받은 트래픽, 리스너에 대한 기본 액션을 설정하는 것이다. 이 부분이 리스너의 `역할`이 된다.   
![image](https://user-images.githubusercontent.com/31242766/210027507-e3f3388c-c29c-4ddd-98fe-7ce895252718.png)   
![image](https://user-images.githubusercontent.com/31242766/210027522-f2e6f9b1-8d80-4ffd-a798-c285515c6ef3.png)   
![image](https://user-images.githubusercontent.com/31242766/210027565-21a20f31-2601-4545-bc13-c7f932f799bf.png)

3. 웹 브라우저를 통한 Application Load Balancer 작동 테스트
![tempsnip](https://user-images.githubusercontent.com/31242766/210027855-fbcd9de4-bba4-47cc-88b2-891b5081afad.png)   
![image](https://user-images.githubusercontent.com/31242766/210028465-f460c381-80f0-4c87-8409-83a858f1cd24.png)   
![image](https://user-images.githubusercontent.com/31242766/210028476-ae48baa7-3c18-41aa-b9e6-3298e33454bc.png)

### Bastion host와 NAT Gateway를 통한 Private EC2 인스턴스의 외부 통신 구성
![image](https://user-images.githubusercontent.com/31242766/209811046-a1fdafc9-b231-43f8-901b-3b9e5f7b9d0f.png)
1. Private subnet에 EC2 생성
- 이름 및 태그, 애플리케이션 및 OS 이미지(Amazon Machine Image)   
![image](https://user-images.githubusercontent.com/31242766/210028802-e7d0dde6-85b0-475d-bfed-57b4053ad81b.png)   
- 키 페어   
![image](https://user-images.githubusercontent.com/31242766/210028972-c0dd76c0-50ef-472f-aeec-3292a1df66ac.png)   
- 네트워크 설정   
![image](https://user-images.githubusercontent.com/31242766/210029179-cd1f8cf0-c993-4b0d-9dc6-46359e3a63ab.png)   
![image](https://user-images.githubusercontent.com/31242766/210029193-6f72f0d4-42a7-4746-8fc0-521f41b5a3d8.png)   
- 스토리지 구성   
![image](https://user-images.githubusercontent.com/31242766/210029221-e3797a1a-20f2-4d0c-a666-6857c0050f8e.png)   
- 고급 세부 정보   
![image](https://user-images.githubusercontent.com/31242766/210029353-43dc5e52-79ce-4761-9d57-3b7ca87201d5.png)   
![image](https://user-images.githubusercontent.com/31242766/210029476-903f7277-e2e2-4396-888b-e1d10884b4bc.png)
![image](https://user-images.githubusercontent.com/31242766/210029517-8a14b74e-66c6-4e82-a7fb-c04f30f1da5d.png)

2. Public subnet의 EC2를 통해 Private subnet의 EC2에 접속 (+ key pair 생성)   
`public-ec2-a1`을 통해서 `private-ec2-a1`에 접속해보자.   
- `private-ec2-a1` 키 페어 파일을 `public-ec2-a1`에 편집기를 이용하여 생성   
![image](https://user-images.githubusercontent.com/31242766/210034562-1a8e6dbc-3f2e-4241-bea9-805c095ab8e3.png)
```linux
[ec2-user@ip-10-1-1-143 ~]$ vi ec2-private-seoul.pem
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAjFRzIXJhJsOqP2BU85QFZKpX4ilbGdqPLJy1k2M6GI56fOvq
aQETFjosvl1Ct8Yg8UpSEXW6UhhtCpuFquZVeAA2CsdfQoFeO9UDnFE+LFEN8y4b
Yfyl52eWTP5LkFg3rCai9LWa5KW3EMwuEHzLHyX3eRpoLKfFBnkK7T/NmyzSzkY7
PqxUw8S/pwzqTsbos14kgajkPphL5x4ohgai2n0VMBFY/hQO3dLfHPpHcIuspuyt
BpmzJj1+6ma8aXIlceIDFbsZFuwcNk2Nk9ghq2Rb1ILx2bZ75aSJ9K4lUyotCrbO
+YsjCG2zW4rA2n12ZmNjM+ehUyQ+BDbT0V6otwIDAQABAoIBAE8ZZxAKVgKwte0l
Up096VBVyFXd89D95khCSQM8IkonPZnerPHlPioAPqpLDUljb7wypVHcJ1sRE52w
DkdHsBOFIvEucl6dZ02Yg4GANehrA874RU0VSTrHo+vgRx2k7DSoTqBbIWUSl77r
KGf4v9Hd58GmhePt6Vv9rJAQr8dwRbFrhlSUDGYVz5XH/ktzWKrdc/UOHc2Vqs0v
wSQTKnGF3AC+xYKgtBOUNOi8kCFC+fwUkToArB/fh6c8XNLWMQxQC8sI/btdt6Xx
gosINhRpBfndqasbI7CwrT3AVtwHVDz7FYie/97tl1p9gnyZGNYoIJUp2G7i35wL
IhkX63kCgYEAxwAmKLDkmVnEN5hkuyfN15MpY9QOTCXB/XPcbN6QL8jdyYsiSE4H
fyRdyNz4YfspIrklml6RDf1GxBKjvBQ4MGxkPZHjNMlsHoPXd+/GiSPEb8J6lawh
8gQCj/T8BmwuNPJCDpiVHKTSYKGLSJnAcuiGqS106/PCg4TYKaopUk0CgYEAtIY7
diE9iq59UJXu0iKWt0jCaM13pbPfA8YdU99CvAXVD7zIDDVKRJZ3S1Atvj/+5eHX
JC9s31cbHY/V4MzYMJ2eCWhIyiAyFY1rY2JsPvgv/lh1H0r+Cre19otyzeUzsrXl
U0S3vCffxoMNGB6eOIScvLR/i6t90bTigaAIQRMCgYBA8NiDEO8Y6EVzSyUcOmof
PqQUMuCTkwLSflvhn2P4ZBmUqvX+GJCzuh9s7EeWWgtbjIYr8U5u/Ud5twd92i9Y
BhdUTGaUFGNXNfk756Cnomd5fULZ0zmkrBBWAEG6qtUNbD2IW9zVYyhQZod4osw9
84n2baIpWfwRRWnxdtlTRQKBgEJ5r4m3gdcAnArBu1jL/d3uQBChoK53Bud327LX
4tYj+6o45R2BviB1m+Yy1zVYkX+LY1Li19+CTuza23JVXELCt8BVE4DCzE6dbe8B
/kRN1jZ90ls8nUHLFol8HkFtZlUnoBPCmToDIOcTuQ2psK+1PZZLjTAcbU98oXAE
QyXbAoGAATP3s6D6ullxY7attcLpwHSotNo56YuEIbpKWU38JZBLrMw8bKORPBgX
SxNygfmsq7RJvAfRCy1YFlq8VQHwOtXqERun7CrlVvOgohsdyHibpiKcQyHs3iyv
v5gSJ8bCo55f3t2wmbSCeLzb7nvq6r7bBTvlXHj8zZdPNG2AB+I=
-----END RSA PRIVATE KEY-----
[ec2-user@ip-10-1-1-143 ~]$ chmod 400 ec2-private-seoul.pem
[ec2-user@ip-10-1-1-143 ~]$ ssh -i ec2-private-seoul.pem ec2-user@private-ec2-a1인스턴스프라이빗IP주소
The authenticity of host '10.1.3.149 (10.1.3.149)' can't be established.
ECDSA key fingerprint is SHA256:5UTQVy4MCxottUBHeyFM8hABxmxqX6lfVhKZfU/JYPg.
ECDSA key fingerprint is MD5:e6:96:25:f1:31:34:61:85:25:a9:1b:53:7c:6e:c7:a9.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.1.3.149' (ECDSA) to the list of known hosts.
Last login: Thu Dec 29 07:33:04 2022 from 221.133.55.118

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-10-1-3-149 ~]$
```
`public-ec2-a1`은 `private-ec2-a1`에 접속하기 위한 중계 서버 역할을 하게 된다. 여기서 `public-ec2-a1`을 `Bestion 호스트`라고 부른다. 
내부 네트워크와 외부 네트워크를 연결하는 일종의 게이트웨이 역할을 수행하는 의미를 가지고 있다. 
`private-ec2-a1`은 외부 인터넷과 직접적인 통신이 불가하다. 하지만 private 네트워크라 하더라도 패키지 설치 등 등 다운로드가 필요한 경우가 생긴다. 
이럴 땐 외부와 통신 경로가 필요하다. `NAT Gateway`를 사용해서 `private-ec2-a1`이 외부와 통신이 가능한 환경을 구성하자.

3. NAT Gateway 생성

4. Route table 설정
5. Private subnet의 EC2의 외부 통신 테스트

### Application Load Balancer를 통한 이중화 네트워크 구성 (2)
![image](https://user-images.githubusercontent.com/31242766/209811216-c4d830c7-7ce8-432a-a709-617f74d87ef3.png)
1. Target group 생성
2. Application Load Balancer 구성
3. 웹 브라우저를 통한 Application Load Balancer 작동 테스트
