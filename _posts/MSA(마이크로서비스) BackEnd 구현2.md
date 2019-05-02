---
layout: post
title:  "MSA(마이크로서비스) BackEnd 구현2"
date:   2019-05-02 21:00:59
author: Jm Park
categories: MSA
tags: msa
---

# Eureka Server란  
MSA 서비스 구축을 하면서 필수적 요소가 되었다.(Netflix제공)  
비즈니스 로직이 위치한 App 서버단의 로드밸런스와 Failover를 위해 서비스를 배치해주는 REST기반 서비스이다.  

# MSA 서비스와 Eureka
1편에서 본 ITEM서비스와 같은 여러 MSA서비스들을 한 파일에 모아둘 경우 Eureka서버를 함께 사용한다.  
이때 netflix-oss(bureau, config, callgate) 를 import하면 된다.  

## 1. 각 MSA 서비스를 Client로 등록
예로 ItemService와 UserService가 있다고 한다.  
각 서비스의 **boot** 의 config 패키지 안에는 ~App.java 파일이 있다.  

이곳에 어노테이션 한줄을 추가한다.  

```{.java}
....
 EnableDiscoveryClient
 public class ItemApp {
     ...
 }
....
```

이후, 각 boot에 있는 pom.xml에 의존성을 추가한다.  
```{.xml}
....
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
....
```

## 2. bootstrap.yml 설정 파일에 Eureka client 등록  
각 MSA 서비스의 **boot**의 src/main/resources/bootstrap.yml에 Eureka client를 등록한다.  

```
eureka:
  client:
    serviceUrl:
      defaultZone: ${EUREKA_URI:http://localhost:8761/eureka}
    enabled: true
```

## 3. 프로젝트 시작 순서
STS의 왼쪽 하단에 Boot Dashboard가 있다.  
그곳에서  
1. config  
2. bereau  
3. item-boot  
4. user-boot  
순으로 시작하면 서비스는 잘 동작하게된다.  
쉽게 생각하면, Eureka서버를 먼저 동작시킨 후 유레카의 클라이언트인 각 MSA서비스를 동작시키는 원리이다.  
