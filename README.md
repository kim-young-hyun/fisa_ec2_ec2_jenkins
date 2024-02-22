# EC2사이 자동 배포

1번-EC2 에서는 jenkins로 build해서 2번-EC2로 내보냄

2번-EC는 받은 파일을 실행

## 1. jenkins 설치

### 환경 구현 

EC2에 build를 위해 docker와 jenkins를 설치한다.

```
$ sudo apt-get update
$ sudo apt install docker.io -y
$ docker run --name myjenkins --privileged -p 8080:8080 jenkins/jenkins:lts-jdk17
```

### docker permission denied

EC2에 docker를 설치했다면 그룹에 추가해야 사용할 수 있다.

```
$ sudo usermod -aG docker $USER
$ newgrp docker
```

## 2. jenkins 설정

### 로그인

jenkins 비밀번호 확인

```
$ docker exec myjenkins sh -c 'cat /var/jenkins_home/secrets/initialAdminPassword' 
```

[http://127.0.0.1:8080으로](http://127.0.0.1:8080으로/) 접속

### 파이프라인 구성

GitHub project 체크, Repository url입력

GitHub hook trigger for GITScm polling 체크

```shell
pipeline {
    agent any
    stages {
        stage('git pull') {
            steps {
                echo 'start pull'
                git branch: '----', credentialsId: '----', url: '----'
                echo 'end pull'
            }
        }
        stage('build') {
            steps {
                echo 'start build'
                sh './gradlew build'
                echo 'end build'
            }
        }
        stage('push') {
            steps {
                echo 'start push'
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: '----',
                            transfers: [
                                sshTransfer(
                                    cleanRemote: false,
                                    excludes: '',
                                    execCommand: 'sh /home/ubuntu/test/----.sh',
                                    execTimeout: 0,
                                    flatten: false,
                                    makeEmptyDirs: false,
                                    noDefaultExcludes: false,
                                    patternSeparator: '[, ]+',
                                    remoteDirectory: 'test/',
                                    remoteDirectorySDF: false,
                                    removePrefix: 'build/libs',
                                    sourceFiles: 'build/libs/*SNAPSHOT.jar'
                                )
                            ],
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: false
                        )
                    ]
                )
                echo 'end push'
            }
        }
    }
}
```

### execCommand

execCommand가 jar 파일 실행에 필요한 shell script를 실행시킨다.

이때 파일 위치는 절대경로로 해주어야 제대로 인식한다.

#### pipeline syntax - sshPublisher

**SSH Server**

1. Name: 사용할 이름 아무거나
2. Transfers
   1. Source files: 생성된 jarfile의 위치 입력 [build/libs/*SNAPSHOT.jar]
   2. Remove prefix: jarfile의 파일위치 입력 [build/libs]
   3. Remote directory: ec2에 jar파일이 올라갈 위치입력 [test/]
   4. Exec command: connection 후 실행될 명령어 [echo "push success"]

## 3. github 설정

### webhook 생성

Settings -> Webhooks -> add webhook

Payload URL: http://<ngrok에 나오는 주소>/github-webhook/

Content type: application/json

## 4. 실행 환경 설정

실행에 필요한 jre를 설치한다.

```
$ sudo apt install openjdk-17-jre-headless
```

### shell script

실행에 필요한 shell script를 작성한다.

```shell
#!/bin/bash

chmod 755 /home/ubuntu/test/*SNAPSHOT.jar
nohup java -jar /home/ubuntu/test/*SNAPSHOT.jar &
```

jar파일 실행은 nohup을 이용해 백그라운드로 실행해야 정상적으로 실행된다.

### port 충돌

특정 port를 사용중인 프로그램 종료

```
$ sudo kill $(sudo lsof -t -i:포트번호)
```

