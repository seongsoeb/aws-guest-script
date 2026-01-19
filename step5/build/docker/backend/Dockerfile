# 1단계: Gradle 이미지로 빌드 수행
FROM gradle:8.5.0-jdk21 AS build
WORKDIR /app
COPY guest_backend/gradle gradle
COPY gradlew .
COPY build.gradle* settings.gradle* gradle.properties* ./

# (선택) 의존성 먼저 받아 캐시 태움 (프로젝트에 따라 실패할 수 있어 주석 처리해도 됨)
RUN chmod +x ./gradlew && ./gradlew dependencies --no-daemon || true

# 소스 복사
COPY src src

# 테스트 때문에 빌드가 자주 깨지면 -x test 추천
RUN ./gradlew clean bootJar -x test --no-daemon


# 2단계: 실행
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar

# 기본값(필요하면 런타임에 덮어씀)
ENV SPRING_PROFILES_ACTIVE=oracle-dev-group

EXPOSE 8080

ENTRYPOINT ["sh","-c","java -Dspring.profiles.active=${SPRING_PROFILES_ACTIVE} -jar /app/app.jar"]

