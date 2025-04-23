---
title: "API 서버의 새로운 시작, 그리고 예상치 못한 장애물"
category: [Troubleshooting]
# tag: [jpa, db, spring boot]
toc: true
toc_sticky: true
published: false
---
## 개요

최근 우리는 API 서버의 새로운 구조를 위해 Java Spring Boot 3.x 버전으로 한 단계 업그레이드된 환경을 구축했습니다. 하지만 기존의 서비스들은 여전히 Spring Boot 2.7.x 버전을 사용하고 있고, 그와 함께 사용하는 JPA 모델 관리 라이브러리는 `javax.persistence` 패키지로 엔티티가 구현되어 있습니다. 새로운 서버에 이 라이브러리를 Maven을 통해 가져와 사용하려는 순간, 예상치 못한 호환성 문제에 직면하게 되었습니다. Spring Boot의 버전 간 변화로 인해 발생한 이 문제는 우리의 개발 과정에서 또 하나의 도전 과제가 되었습니다. 이번 글에서는 이 히스토리를 돌아보며 해결 과정을 공유해 보려 합니다.

---

## 에러 분석의 시작 - 원인을 찾기 위한 첫 발걸음

프로젝트 내부에 직접 `GdpDispmallMEntity` 엔티티를 생성하고, `DpDispMallRepository`를 통해 간단한 조회 테스트를 진행했죠. 아래와 같은 코드로 로컬에서 테스트를 돌렸을 때는 아무런 문제 없이 잘 작동했습니다.

```java
package com.innog.repository.jpa;

import com.innog.model.entity.dp.GdpDispmallMEntity;
import jakarta.transaction.Transactional;
import java.util.List;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@Slf4j
@SpringBootTest(properties = "spring.profiles.active:local")
@Transactional
public class JpaTest {

    @Autowired
    DpDispMallRepository dpDispMallRepository;

    @Test
    void 전시몰_조회_테스트() {
        List<GdpDispmallMEntity> all = dpDispMallRepository.findAll();
        all.forEach(System.out::println);
    }
}
```

테스트는 매끄럽게 통과했습니다. 새로 작성한 엔티티와 리포지토리가 잘 연결되어, 데이터베이스 조회가 정상적으로 이루어지는 걸 확인할 수 있었어요. 이 시점에서는 이 테스트를 더 확장해도 문제없이 잘 돌아갈 거라 생각했죠. 하지만, 분리되어 있던 기존 엔티티와 JPA 모델 라이브러리를 새로운 API 서버에 적용 후에는`Not a managed type: class java.lang.Object`라는 에러가 발생했습니다. 콘솔에는 리포지토리, 서비스, 컨트롤러 등 모든 계층에서 같은 에러가 출력되었습니다.

### 다양한 시도

- **@NoRepositoryBean**을 리포지토리에 선언하기: 실패
- 메인 애플리케이션에 **@ComponentScan, @EntityScan** 등을 사용해 패키지 경로 지정: 실패
- **@EntityScan vs @ComponentScan** 차이 확인
    - **@EntityScan**은 JPA 엔티티를 지정된 패키지에서 찾도록 설정하고, **@ComponentScan**은 Spring Bean으로 등록할 컴포넌트를 스캔하도록 사용됩니다.여러 패키지 구성, 어노테이션 설정 등을 시도해보았으나 문제를 해결할 수 없었습니다.

이후 **@NoBeanRepository** 어노테이션을 삭제하고, Spring Boot 설정도 변경해보았으나 여전히 해결되지 않았습니다. 여러 구조와 설정을 점검하고 Dependency도 확인했으나 실패했습니다.

## **스프링 부트와 JPA 패키지 호환성 이슈**

계속해서 디버깅과 분석 중에 발견한 중요한 힌트는, **스프링 부트와 JPA 패키지 간의 버전 호환성**이 유지되어야 한다는 것이었습니다. 현재 운영 중인 서비스에서 사용하는 모델 라이브러리는 기존의 스프링 부트 2.7.x 버전과 호환되도록 **javax.persistence**로 구현되어 있었습니다. 하지만 이번에 새롭게 구축한 API 서버는 **Spring Boot 3.5.5 버전**을 사용하고 있었고, 이 버전부터는 **jakarta.persistence**로의 전환이 요구되었습니다. 이 점이 문제의 핵심 원인이었죠.

### **스프링 부트와 JPA 호환성 문제: 마이그레이션의 딜레마**

문제의 단서를 찾은 후, 단순히 **jakarta.persistence**로 전환하는 방법을 생각했지만, 여기서 또 다른 난관이 있었습니다. **기존 운영 중인 10개 이상의 애플리케이션**이 여전히 스프링 부트 2.7.x 버전에서 구동 중이었고, 이 애플리케이션들은 모두 `javax.persistence` 기반으로 구축된 동일한 모델 라이브러리를 사용하고 있었습니다. 즉, **모델 라이브러리를 `jakarta.persistence`로 마이그레이션하면 기존 애플리케이션과의 호환성 문제가 발생**하는 상황이었습니다.

### **스프링 부트 3.x와 기존 서비스 호환성 문제: javax.persistence와 jakarta.persistence 전환하기**

마이그레이션 없이 양쪽 환경 모두를 호환시킬 방법이 필요했습니다. 기존 코드에서는 여전히 `javax.persistence`를 사용하되, 빌드된 파일에서는 `jakarta.persistence`로 변환되도록 하는 아이디어를 찾게 되었죠.

### 문제 해결: `maven-shade-plugin`을 사용한 패키지 변환

```java
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.2.4</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <relocations>
                            <relocation>
                                <pattern>javax.persistence</pattern>
                                <shadedPattern>jakarta.persistence</shadedPattern>
                            </relocation>
                        </relocations>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>

```

클래스 파일 내부에서 사용하는 패키지 경로를 자동으로 재배치해주는 플러그인입니다. 이를 통해 코드에서는 `javax.persistence`를 유지하면서, 빌드된 결과물에서는 `jakarta.persistence` 패키지를 사용할  수 있어 보였습니다.
이 설정을 추가한 후, 프로젝트를 빌드하면 `javax.persistence` 패키지를 `jakarta.persistence`로 변환하여 최종 JAR 파일에 저장합니다. 이렇게 하면 다음과 같은 이점이 있어 보였습니다.

- **코드 호환성 유지**: 개발 환경과 기존 2.7.x 버전의 애플리케이션에서는 여전히 `javax.persistence`를 사용하므로, 코드 수정이 최소화됩니다.
- **빌드 결과물의 호환성 확보**: 빌드된 JAR 파일에서는 `jakarta.persistence` 패키지가 사용되므로, 스프링 부트 3.x의 요구사항에 맞춰져 호환성을 유지할 수 있습니다.

이 설정을 통해 `javax.persistence` 패키지를 빌드 시 `jakarta.persistence`로 변환하려 했지만, **소스 빌드 단계에서 여전히 `javax.persistence`를 사용**하는 모델 라이브러리로 인해 에러가 발생했습니다. 이미 Nexus에 `javax.persistence` 패키지로 배포된 모델 라이브러리 자체는 변경할 수 없었기 때문에, 결과적으로는 패키지 재배치 방법을 통해 문제를 해결하는 데 실패했습니다.

스프링 부트 3.x와 JPA 간의 호환성 문제를 해결하기 위해 여러 방법을 시도했지만, **모델 라이브러리가 계속해서 `javax.persistence`를 유지해야 했던 상황**에서는 빌드 에러를 피할 수 없었습니다. 스프링 부트 3.x에서 `jakarta.persistence`로의 전환이 필수인 상황에서는, 이러한 구조적인 제약이 존재할 경우 새로운 해결책이나 전체적인 아키텍처 재검토가 필요하다는 점을 느꼈습니다.

## 결론

최종적으로, 스프링 부트 3.x와 기존의 `javax.persistence` 기반 라이브러리 간 호환성 문제를 해결하기 위해 가능한 모든 실용적인 방법을 시도해 보았습니다.

1. **`spring.jpa.hibernate.use-new-id-generator-mappings=false`** 설정을 통해 Hibernate의 새로운 ID 생성 규칙을 사용하지 않도록 하여 호환성을 확보하려 했습니다.
2. **Eclipse Transformer**와 같은 **변환 라이브러리**를 추가해 `javax.persistence`와 `jakarta.persistence` 간의 변환을 자동화하는 방식을 사용했습니다.
3. **Hibernate 레거시 호환성 모드**를 활성화해, `javax.persistence` 기반 코드가 그대로 작동하도록 설정했습니다.

이러한 다양한 방법을 적용해 보았으나, **완전한 호환성 확보에는 실패**했습니다. 각각의 설정과 라이브러리 적용이 일부 문제를 해결해 주었지만, 여전히 **전체 애플리케이션이 정상적으로 작동하는 데에는 부족**했습니다. `javax.persistence`와 `jakarta.persistence` 간의 근본적인 차이로 인해 예상치 못한 호환성 문제가 계속 발생했습니다.

결국, 새로운 API 서버를 스프링 부트 3.x로 유지하는 대신, 호환성을 위해 버전을 **2.7.x로 낮추는 방법은 선택할 수 없는 상황**이었습니다. 이를 통해 기존 서비스와 완전한 호환성을 확보하는 것이 불가능하다는 결론에 도달했습니다.

따라서, 스프링 부트 3.x와 `javax.persistence` 기반의 레거시 시스템 간에는 호환성 문제가 발생할 수 있으며, 완전한 해결이 필요한 경우 더 근본적인 아키텍처 변경이나 마이그레이션 계획이 필요함을 알게 되었습니다.