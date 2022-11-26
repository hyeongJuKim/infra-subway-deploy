<p align="center">
    <img width="200px;" src="https://raw.githubusercontent.com/woowacourse/atdd-subway-admin-frontend/master/images/main_logo.png"/>
</p>
<p align="center">
  <img alt="npm" src="https://img.shields.io/badge/npm-%3E%3D%205.5.0-blue">
  <img alt="node" src="https://img.shields.io/badge/node-%3E%3D%209.3.0-blue">
  <a href="https://edu.nextstep.camp/c/R89PYi5H" alt="nextstep atdd">
    <img alt="Website" src="https://img.shields.io/website?url=https%3A%2F%2Fedu.nextstep.camp%2Fc%2FR89PYi5H">
  </a>
  <img alt="GitHub" src="https://img.shields.io/github/license/next-step/atdd-subway-service">
</p>

<br>

# 인프라공방 샘플 서비스 - 지하철 노선도

<br>

## 🚀 Getting Started

### Install
#### npm 설치
```
cd frontend
npm install
```
> `frontend` 디렉토리에서 수행해야 합니다.

### Usage
#### webpack server 구동
```
npm run dev
```
#### application 구동
```
./gradlew clean build
```
<br>

## 미션

* 미션 진행 후에 아래 질문의 답을 README.md 파일에 작성하여 PR을 보내주세요.

### 0단계 - pem 키 생성하기

1. 서버에 접속을 위한 pem키를 [구글드라이브](https://drive.google.com/drive/folders/1dZiCUwNeH1LMglp8dyTqqsL1b2yBnzd1?usp=sharing)에 업로드해주세요

2. 업로드한 pem키는 무엇인가요.
- hyeongjukim.pem

### 1단계 - 망 구성하기
1. 구성한 망의 서브넷 대역을 알려주세요
- 대역
  - public 192.168.23.0/26
  - public 192.168.23.64/26
  - internal 192.168.23.128/27
  - admin 192.168.23.160/27

2. 배포한 서비스의 공인 IP(혹은 URL)를 알려주세요
- URL : 
  - 3.35.148.36
  - hyeongjukim.o-r.kr
  
3. 요구사항
- [x] VPC 생성
  - [x] CIDR은 C class(x.x.x.x/24)로 생성. 이 때, 다른 사람과 겹치지 않게 생성
- [x] Subnet 생성
  - [x] 외부망으로 사용할 Subnet : 64개씩 2개 (AZ를 다르게 구성)
  - [x] 내부망으로 사용할 Subnet : 32개씩 1개
  - [x] 관리용으로 사용할 Subnet : 32개씩 1개
- [x] Internet Gateway 연결
- [x] Route Table 생성
- [x] Security Group 설정
  - [x] 외부망
    - [x] 전체 대역 : 8080 포트 오픈
    - [x] 관리망 : 22번 포트 오픈
  - [x] 내부망
    - [x] 외부망 : 3306 포트 오픈
    - [x] 관리망 : 22번 포트 오픈
  - [x] 관리망
    - [x] 자신의 공인 IP : 22번 포트 오픈
- [x] 서버 생성
  - [x] 외부망에 웹 서비스용도의 EC2 생성
  - [x] 내부망에 데이터베이스용도의 EC2 생성
  - [x] 관리망에 베스쳔 서버용도의 EC2 생성
  - [x] 베스쳔 서버에 Session Timeout 600s 설정
  - [x] 베스쳔 서버에 Command 감사로그 설정
- [x] 외부망에 웹 애플리케이션을 배포
- [x] DNS 설정

---

### 2단계 - 배포하기
1. TLS가 적용된 URL을 알려주세요

- URL : 
  - 3.35.148.36
  - hyeongjukim.o-r.kr
- 운영 환경 구성하기
  - [x] 외부망에 docker 설치
  - [x] 외부망에 Nginx로 Reverse Proxy를 구성
  - [x] Reverse Proxy에 TLS 설정
  - [x] 운영 데이터베이스 구성하기
- 개발 환경 구성하기
- [x] 설정 파일 나누기
- [x] JUnit : h2, Local : docker(mysql), Prod : 운영 DB를 사용하도록 설정
- [x] submodule을 사용하여 설정파일 관리하기
- [x] 로컬테스트(MultiRun)

---

### 3단계 - 배포 스크립트 작성하기

1. 작성한 배포 스크립트를 공유해주세요.


```shell
#!/bin/bash
REPOSITORY=~/nextstep/infra-subway-deploy
BRANCH=step3-deploy-script
APPLICATION_NAME=subway
LOG_FILE_NAME=web-service.log

txtrst='\033[1;37m' # White
txtred='\033[1;31m' # Red
txtylw='\033[1;33m' # Yellow
txtpur='\033[1;35m' # Purple
txtgrn='\033[1;32m' # Green
txtgra='\033[1;30m' # Gray

echo -e "${txtylw}=======================================${txtrst}"
echo -e "${txtgrn}  << 스크립트 🧐 >>${txtrst}"
echo -e "${txtylw}=======================================${txtrst}"

function check_diff() {
    git fetch
    master=$(git rev-parse $BRANCH)
    remote=$(git rev-parse origin/$BRANCH)
    if [[ $master == $remote ]]; then
        echo -e "[$(date)] 변경된 내용이 없습니다. 😫"
        exit 0
    fi
}

function pull() {
    echo "> 🚀 repository pull을 수행합니다."
    git pull origin $BRANCH
    #git submodule update --recursive --remote
}

function build_anc_copy {
    echo "> 🚀 gradle build를 수행합니다."
    ./gradlew clean build
    echo "> 🚀 빌드 파일을 복사합니다."
    cp $REPOSITORY/build/libs/$APPLICATION_NAME*.jar $REPOSITORY
}

function stop_service {
    CURRENT_PID=$(pgrep -f $APPLICATION_NAME.*.jar)
    if [ -z "$CURRENT_PID" ]; then
        echo " 🚀 현재 구동중인 애플리케이션이 없으로 종료하지 않습니다."
    else
        echo " 🚀 현재 구동중인 애플리케이션을 종료합니다."
        echo " 🚀 kill -15 $CURRENT_PID"
        kill -15 $CURRENT_PID
        sleep 5
    fi
}

function start_service() {
        echo "> 🚀 새로운 애플리케이션을 구동합니다."
        nohup java -Djava.security.egd=file:/dev/./urandom -Dserver.port=8080 -jar -Dspring.profiles.active=prod $REPOSITORY/build/libs/subway-0.0.1-SNAPSHOT.jar 1> $LOG_FILE_NAME 2>&1 &
}


cd $REPOSITORY
check_diff
pull
build_anc_copy
stop_service
start_service
```
