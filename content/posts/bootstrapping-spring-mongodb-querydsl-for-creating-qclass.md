---
title: Spring MongoDB에서 QueryDSL QClass 만들기
date: 2021-08-15T10:59:17.118Z
draft: false
description: Spring Boot + MongoDB에서 org.springframework.data.mongodb.core.mapping.Document 애노테이션이 붙은 클래스로 QClass를 만들어보기
tag:
  - spring
  - spring-boot
  - mongodb
  - querydsl
  - kotlin
  - gradle
---

Spring에서 MongoDB와 QueryDSL을 같이 사용하는 예제를 검색해보면 QClass를 생성하기 위해, 위 설정처럼 jpa 설정을 찾아서 사용하는 예제들이 많아 참 안타까웠다.

@Document가 붙은 클래스들만으로도 QClass를 만들 수 있는 쉬운 방법을 공유하고자 한다.


> 예시 코드는 모두 Kotlin으로 작성되었다.

아래 제시한 방법은 다음 두 환경에서 적용해 정상 작동함을 확인했다.

1.
   - kotiln: 1.3.72
   - jdk: 11
   - spring boot: 2.3.3
   - querydsl: 4.3.1
2.
   - kotlin: 1.4.30
   - jdk: 11
   - spring boot: 2.4.9
   - querydsl: 4.4.0


```kotlin
// build.gradle.kts

plugins {
    val kotlinVersion = "1.3.72"
    id("org.springframework.boot") version "2.3.3.RELEASE"
    id("io.spring.dependency-management") version "1.0.10.RELEASE"
    kotlin("jvm") version kotlinVersion
    kotlin("plugin.spring") version kotlinVersion
    // (1)
    kotlin("kapt") version kotlinVersion
}

/*
 * (...)
 */

dependencies {
    /*
     * (...)
     */
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")

    // (2)
    val querydslVersion = "4.3.1"
    implementation("com.querydsl:querydsl-apt:$querydslVersion")
    implementation("com.querydsl:querydsl-mongodb:$querydslVersion")
    kapt("com.querydsl:querydsl-apt")
}

// (3)
kapt {
    annotationProcessor("org.springframework.data.mongodb.repository.support.MongoAnnotationProcessor")
}
/*
 * (...)
 */

```

(1) Kotlin에서 annotation processing을 사용하기 위한 플러그인이다.

(2) mongodb에서 QueryDSL을 사용하기 위한 의존성이다.

(3) 이것이 @Document 애노테이션이 달린 클래스를 QClass로 만들어주는 핵심이라고 볼 수 있는데, 해당 클래스는 다음과 같다.

```java
package org.springframework.data.mongodb.repository.support;

// import ...

/**
 * Annotation processor to create Querydsl query types for QueryDsl annotated classes.
 *
 * @author Oliver Gierke
 * @author Owen Q
 */
@SupportedAnnotationTypes({ "com.querydsl.core.annotations.*", "org.springframework.data.mongodb.core.mapping.*" })
@SupportedSourceVersion(SourceVersion.RELEASE_6)
public class MongoAnnotationProcessor extends AbstractQuerydslProcessor {

	/*
	 * (non-Javadoc)
	 * @see com.querydsl.apt.AbstractQuerydslProcessor#createConfiguration(javax.annotation.processing.RoundEnvironment)
	 */
	@Override
	protected Configuration createConfiguration(@Nullable RoundEnvironment roundEnv) {

		processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "Running " + getClass().getSimpleName());

		DefaultConfiguration configuration = new DefaultConfiguration(processingEnv, roundEnv, Collections.emptySet(),
				QueryEntities.class, Document.class, QuerySupertype.class, QueryEmbeddable.class, QueryEmbedded.class,
				QueryTransient.class);
		configuration.setUnknownAsEmbedded(true);

		return configuration;
	}
}
```

`@Document` annotation이 달린 class를 QueryDSL의 entity 클래스로 볼 수 있도록 `DefaultConfiguration` 객체를 생성한다.

결국 핵심은 `org.springframework.data.mongodb.repository.support.MongoAnnotationProcessor`를 이용해서 QClass를 만드는 것이기 때문에 Java, Maven, groovy(build.gradle)를 사용하는 분들은 각자의 환경에서 `MongoAnnotationProcessor`를 사용하는 방법을 찾아 적용하면 된다.
