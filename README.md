
# 서킷 브레이커 구현 레파지토리

#### 💡 상품을 조회하는 프로젝트를 가정합니다.
1. 상품 아이디 111을 호출하면 에러를 발생시켜 `fallbackMethod`를 실행하는 것을 확인합니다.
   (fallbackMethod 실행된 뒤 일정시간 이후 다시 fallbackMethod가 해제되는 것을 볼 수 있습니다.) 

    <img width="419" alt="image" src="https://github.com/user-attachments/assets/b90c1c5a-1281-4ca6-a076-8b29fa3d54c1">

2. actuator/prometheus 로그 조회
![image](https://github.com/user-attachments/assets/f51ceb15-2dc7-4466-a334-f5a9da73660e)
 ( "closed" ~ "half_open" ~ "open" )

---

## 1. 서킷 브레이커 (Resilience4j)

### 1.1 서킷 브레이커 개요

### 1.1.1 서킷 브레이커란?

- 서킷 브레이커는 마이크로서비스 간의 호출 실패를 감지하고 시스템의 전체적인 안정성을 유지하는 패턴
- 외부 서비스 호출 실패 시 빠른 실패를 통해 장애를 격리하고, 시스템의 다른 부분에 영향을 주지 않도록 합니다.
- 상태 변화: 클로즈드 -> 오픈 -> 하프-오픈

## 1.2 Resilience4j 개요

### 1.2.1 Resilience4j란?

- Resilience4j는 서킷 브레이커 라이브러리로, 서비스 간의 호출 실패를 감지하고 시스템의 안정성을 유지합니다.
- 다양한 서킷 브레이커 기능을 제공하며, 장애 격리 및 빠른 실패를 통해 복원력을 높입니다.

### 1.2.2 Resilience4j의 주요 특징

- **서킷 브레이커 상태**: 클로즈드, 오픈, 하프-오픈 상태를 통해 호출 실패를 관리
    - **클로즈드(Closed)**:
        - 기본 상태로, 모든 요청을 통과시킵니다.
        - 이 상태에서 호출이 실패하면 실패 카운터가 증가합니다.
        - 실패율이 설정된 임계값(예: 50%)을 초과하면 서킷 브레이커가 오픈 상태로 전환됩니다.
        - 예시: 최근 5번의 호출 중 3번이 실패하여 실패율이 60%에 도달하면 오픈 상태로 전환됩니다.
    - **오픈(Open)**:
        - 서킷 브레이커가 오픈 상태로 전환되면 모든 요청을 즉시 실패로 처리합니다.
        - 이 상태에서 요청이 실패하지 않고 바로 에러 응답을 반환합니다.
        - 설정된 대기 시간이 지난 후, 서킷 브레이커는 하프-오픈 상태로 전환됩니다.
        - 예시: 서킷 브레이커가 오픈 상태로 전환되고 20초 동안 모든 요청이 차단됩니다.
    - **하프-오픈(Half-Open)**:
        - 오픈 상태에서 대기 시간이 지나면 서킷 브레이커는 하프-오픈 상태로 전환됩니다.
        - 하프-오픈 상태에서는 제한된 수의 요청을 허용하여 시스템이 정상 상태로 복구되었는지 확인합니다.
        - 요청이 성공하면 서킷 브레이커는 클로즈드 상태로 전환됩니다.
        - 요청이 다시 실패하면 서킷 브레이커는 다시 오픈 상태로 전환됩니다.
        - 예시: 하프-오픈 상태에서 3개의 요청을 허용하고, 모두 성공하면 클로즈드 상태로 전환됩니다. 만약 하나라도 실패하면 다시 오픈 상태로 전환됩니다.
- **Fallback**: 호출 실패 시 대체 로직을 제공하여 시스템 안정성 확보
- **모니터링**: 서킷 브레이커 상태를 모니터링하고 관리할 수 있는 다양한 도구 제공

## 1.3 Fallback 메커니즘

### 1.3.1 Fallback 설정

- Fallback 메서드는 외부 서비스 호출이 실패했을 때 대체 로직을 제공하는 메서드입니다.

### 1.3.2 Fallback의 장점

- 시스템의 안정성을 높이고, 장애가 발생해도 사용자에게 일정한 응답을 제공할 수 있습니다.
- 장애가 다른 서비스에 전파되는 것을 방지합니다.

## 1.4 Resilience4j Dashboard

### 1.4.1 Resilience4j Dashboard 설정

- Resilience4j Dashboard를 사용하여 서킷 브레이커의 상태를 모니터링할 수 있습니다.
- `build.gradle` 파일 예시:
    
    ```groovy
    
    dependencies {
        implementation 'io.github.resilience4j:resilience4j-micrometer'
        implementation 'io.micrometer:micrometer-registry-prometheus'
        implementation 'org.springframework.boot:spring-boot-starter-actuator'
    }
    
    ```
    
- 설정은 `application.yml` 파일에서 설정할 수 있습니다.
- 예시 코드:
    
    ```java
    management:
      endpoints:
        web:
          exposure:
            include: prometheus
      prometheus:
        metrics:
          export:
            enabled: true
    ```
    
- http://${hostname}:${port}/actuator/prometheus 에 접속하여 서킷브레이커 항목을 확인 가능합니다.

### 1.4.2 Resilience4j Dashboard 사용

- Prometheus와 Grafana를 사용하여 Resilience4j 서킷 브레이커의 상태를 실시간으로 모니터링할 수 있습니다.
- Prometheus를 통해 수집된 메트릭을 Grafana 대시보드에서 시각화할 수 있습니다.

## 1.5 Resilience4j와 Spring Cloud 연동

### 1.5.1 Spring Cloud와의 통합

- Resilience4j는 Spring Cloud Netflix 패키지의 일부로, Eureka와 Ribbon 등 다른 Spring Cloud 구성 요소와 쉽게 통합할 수 있습니다.
- Spring Cloud의 서비스 디스커버리와 로드 밸런싱을 활용하여 더욱 안정적인 마이크로서비스 아키텍처를 구축할 수 있습니다.

