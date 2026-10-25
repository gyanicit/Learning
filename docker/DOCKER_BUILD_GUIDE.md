# Docker Build Guide for a Spring Boot Demo App

This guide shows the complete Dockerfile and walks through the image build cycle: preparing the project, building the final image, optionally building the Maven stage for inspection, running the app, and cleaning up.

## 1. Project layout

Run Docker commands from the project root: the directory containing `Dockerfile`, `pom.xml`, and `src/`. A typical Maven Spring Boot project looks like this:

```text
your-project/
├── Dockerfile
├── .dockerignore
├── pom.xml
└── src/
    ├── main/
    │   ├── java/
    │   │   └── com/demo/
    │   │       ├── DemoApplication.java
    │   │       └── GreetingController.java
    │   └── resources/
    │       └── application.properties
    └── test/
```

The commands below work from any folder. Open PowerShell, Command Prompt, or a terminal in the project root. If you need to change directories first, replace the example path with your own project location:

```powershell
Set-Location "C:\path\to\your-project"
```

In macOS or Linux, the equivalent is:

```sh
cd /path/to/your-project
```

The final dot (`.`) in `docker build` means “use the current directory as the build context,” so running the command from the project root is important.

### Optional: minimal sample application files

If you already have the demo project, use its existing `pom.xml` and `src/` files. To create a minimal sample from scratch, make the following files under the project root. This sample uses the same artifact ID, version, Java version, package, and endpoint expected by the Dockerfile.

`pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.1.1</version>
        <relativePath/>
    </parent>

    <groupId>com.demo</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webmvc</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

`src/main/java/com/demo/DemoApplication.java`:

```java
package com.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

`src/main/java/com/demo/GreetingController.java`:

```java
package com.demo;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1")
public class GreetingController {
    @GetMapping("/diwali")
    public ResponseEntity<String> greetForDiwali() {
        return ResponseEntity.ok("Happy Diwali");
    }
}
```

`src/main/resources/application.properties` (optional):

```properties
spring.application.name=demo
```

The Maven project must be in the same directory as the `Dockerfile`. The Java files must be under `src/main/java`; otherwise Maven will not compile them into the JAR.
## 2. Project values used by this Dockerfile

The Dockerfile builds with Java 17 and expects Maven to create this JAR:

```text
target/demo-0.0.1-SNAPSHOT.jar
```

Those values come from the Maven project metadata: Java version `17`, artifact ID `demo`, and version `0.0.1-SNAPSHOT`. If you change the artifact ID or version in `pom.xml`, update the JAR path in the Dockerfile's `COPY --from=build` instruction to match.

The example application exposes `GET /api/v1/diwali` and returns `Happy Diwali`. The controller can look like this:

```java
@RestController
@RequestMapping("/api/v1")
public class GreetingController {

    @GetMapping("/diwali")
    public ResponseEntity<String> greetForDiwali() {
        return ResponseEntity.ok("Happy Diwali");
    }
}
```

The Spring Boot application class starts the web server. The existing Maven project must include the Spring Boot web dependency and Java source files under `src/main` for the Docker build to package the app.

## 3. Complete Dockerfile

Place this file named `Dockerfile` in the project root, alongside `pom.xml`:

```dockerfile
# Build the application with Maven and Java 17.
FROM maven:3.9-eclipse-temurin-17 AS build

# Use a dedicated directory for the Maven build.
WORKDIR /workspace

# Copy the Maven project definition first so dependency downloads can be reused between builds.
COPY pom.xml ./

# Download the project's dependencies before copying source files.
RUN mvn -B -ntp dependency:go-offline

# Copy the application source code into the build stage.
COPY src/ ./src/

# Package the Spring Boot application as an executable JAR; tests are not run during image builds.
RUN mvn -B -ntp -DskipTests package

# Use a Java 17 runtime image for the finished application.
FROM eclipse-temurin:17-jre AS runtime

# Keep the application files together under /app.
WORKDIR /app

# Create an unprivileged account so the application does not run as root.
RUN groupadd --system spring && useradd --system --gid spring --home-dir /app spring

# Copy the executable JAR from the build stage and make the runtime user its owner.
COPY --from=build --chown=spring:spring /workspace/target/demo-0.0.1-SNAPSHOT.jar ./app.jar

# Run the application as the unprivileged account.
USER spring

# Document the port used by Spring Boot's default HTTP server.
EXPOSE 8080

# Start the Spring Boot application when the container launches.
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

## 4. What the Dockerfile does

### Build stage

`FROM maven:3.9-eclipse-temurin-17 AS build` starts the stage named `build`. The image supplies Maven and a Java 17 development kit (JDK), which are needed to compile and package the project. The alias `build` is used later by `COPY --from=build` and by `docker build --target build`.

`WORKDIR /workspace` sets the current directory inside the build stage. `COPY pom.xml ./` copies the Maven project definition there. Docker processes the POM before source files so dependency downloads can remain cached when only source code changes.

`RUN mvn -B -ntp dependency:go-offline` downloads Maven dependencies. `-B` means batch/non-interactive mode; `-ntp` turns off transfer progress output.

`COPY src/ ./src/` copies application code and resources into the build stage. `RUN mvn -B -ntp -DskipTests package` compiles and packages the executable JAR under `/workspace/target`. `-DskipTests` skips running tests during this Docker image build.

### Runtime stage

`FROM eclipse-temurin:17-jre AS runtime` starts a fresh stage with Java 17's runtime environment (JRE). Maven and the source tree from the build stage are not automatically included in this stage.

`WORKDIR /app` sets the app directory. `RUN groupadd ... && useradd ...` creates a non-root `spring` account. `COPY --from=build ...` brings only the packaged JAR from the build stage into the runtime image and sets its owner. `USER spring` runs the app without root privileges.

`EXPOSE 8080` documents the port used inside the container; it does not publish that port on the host. The `-p` option on `docker run` publishes it. `ENTRYPOINT` starts Java with the packaged JAR when a container is launched.

This is a multi-stage build: Maven and the JDK are used to create the JAR in `build`; the final image is based on the JRE `runtime` stage and receives the JAR with `COPY --from=build`.

## 5. Build the final image

From the project root, run:

```sh
docker build -t demo-app .
```

Command parts:

- `docker build` asks Docker to execute the Dockerfile and create an image.
- `-t demo-app` gives the resulting image the name `demo-app`. Since no tag is specified, Docker uses `latest`; this is equivalent to `-t demo-app:latest`.
- `.` is the build context: the current directory. Docker reads the `Dockerfile` there and makes files available to `COPY`, after applying `.dockerignore`.

With no `--target`, Docker builds through to the last stage, `runtime`. It first builds the `build` stage because the runtime stage copies the JAR from it. The result is the runnable image `demo-app:latest`.

The stage alias `build` and image tag `demo-app:latest` are different things. The Dockerfile uses `build` to refer to a stage within one build. The `-t` option assigns a name to the output image.

### Show the build steps

```sh
docker build --progress=plain -t demo-app:latest .
```

`--progress=plain` prints the build steps as text. Steps are labeled by stage (for example, `build` or `runtime`). A `CACHED` step means Docker reused a prior result.

To rerun build instructions without reusing cached steps:

```sh
docker build --no-cache --progress=plain -t demo-app:latest .
```

This can take longer because dependencies may need to be downloaded again. Use it when you want to see a fresh build or investigate whether cached steps are involved.

## 6. Optionally build only the Maven stage

The final image build already uses the Maven stage. This command is optional; it creates an additional named image from the stage so you can inspect it:

```sh
docker build --target build --progress=plain -t demo-app-build:latest .
```

- `--target build` tells Docker to stop at the stage aliased `build`.
- `-t demo-app-build:latest` tags that stage's output as a separate image.
- The image includes Maven, the JDK, copied project files, downloaded dependencies, and the packaged JAR. It is useful for debugging or inspecting the build environment.

The `demo-app-build:latest` tag is not used by `docker build -t demo-app .`. The full build refers directly to the `build` stage in the Dockerfile and then creates the `runtime` stage. Docker may reuse cached steps between builds, but the separate tag does not connect the two commands.

## 7. List and inspect images

List locally stored images:

```sh
docker image ls
```

After the full build, look for repository `demo-app` with tag `latest`. After the optional target build, look for `demo-app-build` with tag `latest`.

To filter by image name:

```sh
docker image ls demo-app
docker image ls demo-app-build
```

To include dangling and intermediate image records:

```sh
docker image ls -a
```

Some builder cache is not shown as a named image. The tags created by the commands above are the easiest outputs to identify.

### Inspect the build-stage files

Start a temporary shell in `demo-app-build`:

```sh
docker run --rm -it --entrypoint /bin/sh demo-app-build:latest
```

`--entrypoint /bin/sh` starts a shell instead of the image's normal Maven entrypoint. `-it` gives an interactive terminal. `--rm` removes the temporary container on exit; the image stays available.

Inside the container, run:

```sh
pwd
ls -la /workspace
ls -la /workspace/src
ls -la /workspace/target
```

The packaged JAR should be in `/workspace/target`. Type `exit` to leave the shell.

## 8. Run and verify the final image

Run in the foreground:

```sh
docker run --rm -p 8080:8080 demo-app:latest
```

Or run in the background:

```sh
docker run -d --name demo-app-container -p 8080:8080 demo-app:latest
```

The format is `docker run [OPTIONS] IMAGE [COMMAND] [ARGUMENTS]`. Put options before the image name. Here, `-d` runs detached, `--name` assigns a container name, and `-p 8080:8080` maps host port 8080 to container port 8080 (`host:container`).

Open this URL on the same computer:

```text
http://localhost:8080/api/v1/diwali
```

The route is spelled `diwali`; the expected response is `Happy Diwali`.

For a detached container, view application output with:

```sh
docker logs -f demo-app-container
```

Stop it with:

```sh
docker stop demo-app-container
```

Stopping a container does not remove its image. To remove the stopped container:

```sh
docker rm demo-app-container
```

## 9. End-to-end command sequence

Run these commands from the project root. The target build and shell inspection are optional; the final image build is sufficient to build and run the application.

```sh
# Build the final runtime image and show stage progress.
docker build --progress=plain -t demo-app:latest .

# Optional: build only the Maven stage as a separately tagged image.
docker build --target build --progress=plain -t demo-app-build:latest .

# List the images.
docker image ls

# Optional: inspect the Maven stage's files.
docker run --rm -it --entrypoint /bin/sh demo-app-build:latest

# Start the final application in the background.
docker run -d --name demo-app-container -p 8080:8080 demo-app:latest

# Follow logs, then stop the container when finished.
docker logs -f demo-app-container
docker stop demo-app-container
```

## Further reading

- [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker image ls reference](https://docs.docker.com/reference/cli/docker/image/ls/)
- [Docker build cache](https://docs.docker.com/build/cache/)


