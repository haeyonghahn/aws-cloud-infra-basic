# aws-cloud-infra-basic
## 목차
* **[EC2 Instance 접속을 위한 PuTTY 사용 방법](#EC2-Instance-접속을-위한-PuTTY-사용-방법)**
* **[서버리스 정적 웹사이트 호스팅 및 성능 가속화](#서버리스-정적-웹사이트-호스팅-및-성능-가속화)**
  * **[아키텍처 관련 기술/서비스/다이어그램/구현 순서 검토](#아키텍처-관련-기술서비스다이어그램구현-순서-검토)**
  * **[S3 Bucket 생성 및 정적 웹사이트 호스팅](#S3-Bucket-생성-및-정적-웹사이트-호스팅)**
  * **[CloudFront를 통한 웹사이트 성능 가속화](#CloudFront를-통한-웹사이트-성능-가속화)**

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
2. Object(File) 업로드
   - mycar.html
   - car.jpg
3. 정적 웹 사이트 호스팅 기능 활성화
4. Bucket과 Object에 대한 액세스 정책 설정
5. 웹 브라우저에서 웹 사이트 작동 확인

### CloudFront를 통한 웹사이트 성능 가속화
1. CloudFront Distribution 생성
   - Origin, Cache behavior 등 설정
2. 웹 브라우저에서 CloudFront Distribution 작동 확인
3. 웹 사이트 성능 테스트
   - S3 정적 웹 사이트 호스팅을 통한 콘텐츠 로드 속도
   - CloudFront를 통한 콘텐츠 로드 속도
