# syntax=docker/dockerfile:1
#
# Production image for JPetStore (DevOps layer added by ElloughWassim).
# The root-level Dockerfile is the upstream MyBatis development image and is kept unchanged.
#
# Build from the repository root:
#   docker build -f deploy/docker/Dockerfile -t jpetstore .

# ---- Build stage -------------------------------------------------------------
FROM eclipse-temurin:21-jdk-noble AS build
WORKDIR /workspace
COPY mvnw pom.xml LICENSE_HEADER format.xml ./
COPY .mvn .mvn
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B -ntp dependency:go-offline -Dlicense.skip=true
COPY src src
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B -ntp package -DskipTests -Dlicense.skip=true
WORKDIR /workspace/target/exploded
RUN jar -xf ../jpetstore.war

# ---- Runtime stage: Tomcat 9 (javax.servlet), JRE 21, non-root ------------------
FROM tomcat:9.0-jre21-temurin-noble
RUN groupadd --system --gid 10001 app \
    && useradd --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app \
    && rm -rf "${CATALINA_HOME}/webapps/"* "${CATALINA_HOME}/webapps.dist" \
    && chown -R 10001:10001 "${CATALINA_HOME}/work" "${CATALINA_HOME}/temp" "${CATALINA_HOME}/logs"
# Deployed as an exploded directory so webapps/ can stay read-only at runtime.
COPY --from=build --chown=10001:10001 /workspace/target/exploded/ ${CATALINA_HOME}/webapps/jpetstore/
USER 10001:10001
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
    CMD ["bash", "-c", "exec 3<>/dev/tcp/127.0.0.1/8080"]
CMD ["catalina.sh", "run"]
