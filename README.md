# 📦 클라우드 보안 아키텍처 관점에서의 APT Proxy 연동 EC2, MySQL, Spring Boot 통합 실습 가이드

<br>

## 👨‍👩‍👧‍👦 Contributors

| <img src="https://github.com/kcs19.png" width="200px"> | <img src="https://github.com/HongChan1412.png" width="200px"> | <img src="https://github.com/letmeloveyou82.png" width="200px"> | <img src="https://github.com/nanahj.png" width="200px"> |
| :---: | :---: | :---: | :---: |
| [김창성](https://github.com/kcs19) | [나홍찬](https://github.com/HongChan1412) | [최윤정](https://github.com/letmeloveyou82) | [이현정](https://github.com/nanahj) |

<br>

## ✅ APT 프록시 기반 EC2 네트워크 아키텍처 구성 환경 시나리오

- **Private EC2**는 외부 인터넷에 직접 연결할 수 없음.
- **Public EC2**는 인터넷에 연결되어 있음.
- `apt-cacher-ng`를 이용해 Public EC2를 **APT Proxy (미러 서버)** 로 설정.
- Private EC2는 Public EC2를 경유하여 `apt update`, `apt install` 등을 실행.
- Spring Boot 앱은 Private EC2의 MySQL에 접근하도록 구성.

---

## 📍 Architecture
![image](https://github.com/user-attachments/assets/e7966ec8-04a7-4330-b01d-8b25d7c6f0ca)


<br>

## 🌐 Public EC2 설정 (`APT 프록시 서버`)
```shell
# 패키지 목록 업데이트 및 apt 프록시 서버 설치
sudo apt update
sudo apt install apt-cacher-ng -y

# Private EC2로 접속
ssh -i "<키페어 명>" ubuntu@<PRIVATE_EC2_PRIVATE_IP>
```
> [!NOTE]  
> Public EC2의 보안 그룹 인바운드에 3142 포트 허용

<br>

## 🔒 Private EC2 설정
```shell
# Public EC2를 프록시로 등록
echo 'Acquire::http::Proxy "http://<PUBLIC_EC2_PRIVATE_IP>:3142";' | sudo tee /etc/apt/apt.conf.d/01proxy

# apt 명령 테스트
sudo apt update
sudo apt install mysql-server -y
```

<br>

## 🛠️ MySQL 설정 (`Private EC2`)
```shell
# MySQL 접속
sudo mysql -u root -p
```
```sql
-- DB 및 사용자 생성
CREATE DATABASE fisa;
CREATE USER 'user01'@'localhost' IDENTIFIED BY 'user01';
CREATE USER 'user01'@'%' IDENTIFIED BY 'user01';
GRANT ALL PRIVILEGES ON fisa.* TO 'user01'@'localhost';
GRANT ALL PRIVILEGES ON fisa.* TO 'user01'@'%';
FLUSH PRIVILEGES;
```
```shell
# 외부 접속 허용 설정
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
# bind-address 항목 수정
bind-address = 0.0.0.0

# MySQL 재시작
sudo systemctl restart mysql.service
```

> [!NOTE]  
> Private EC2의 보안 그룹 인바운드에 3306 포트 허용

<br>

## ⚙️ Spring Boot 설정 (`application.properties`)
```properties
spring.datasource.url=jdbc:mysql://<PRIVATE_EC2_PRIVATE_IP>:3306/fisa?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
spring.datasource.username=user01
spring.datasource.password=user01
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
spring.jpa.hibernate.ddl-auto=create 
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

spring.application.name=step06_SpringDataJPA
server.port=8080
```

<br>

## 🚀 Public EC2에서 Spring Boot 실행
```shell
# JDK 설치
sudo apt install openjdk-17-jre-headless -y

# JAR 파일 실행
java -jar step06_SpringDataJPA-0.0.1-SNAPSHOT.jar
```

> [!NOTE]  
> Public EC2의 보안 그룹 인바운드에 8080 포트 허용 <br>
> `http://<PUBLIC_EC2_PUBLIC_IP>:8080`로 접속 확인
