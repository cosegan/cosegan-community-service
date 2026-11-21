# Shopizer: Docker image 3.2.7 stops at start-up: database.properties not found

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the published Docker image stops at start-up; no money is involved.

**Environment**

- Docker image `shopizerecomm/shopizer:3.2.7` (digest `sha256:199e04c1bce3…`, pushed 2025-03-22, the newest tag), started with its own command, `java -jar /opt/app/shopizer.jar`
- The code cited below is the default branch `3.2.7` (head `6a4a0a6`)
- Docker on macOS (the image runs under amd64 emulation)

**We verified this on our sandbox only**: one machine with Docker on macOS. We have not run the image anywhere else. The README's own example uses `shopizerecomm/shopizer:latest`, an older image (2022-09-05), which we did not try.

**What we are seeing**

`docker run shopizerecomm/shopizer:3.2.7` stops during start-up and exits with code 1:

```
ERROR 1 --- [           main] o.s.boot.SpringApplication               : Application run failed
org.springframework.beans.factory.BeanInitializationException: Could not load properties; nested exception is java.io.FileNotFoundException: class path resource [database.properties] cannot be opened because it does not exist
```

1. No Spring profile is set (the image's command is `java -jar /opt/app/shopizer.jar`, as in `sm-shop/Dockerfile`), so the `default` profile of `sm-core/src/main/resources/spring/shopizer-core-config.xml` applies (lines 78 to 91). It loads `classpath:database.properties` (line 85).
2. The jar has no `database.properties` at the root of its class path, only under `BOOT-INF/classes/profiles/<name>/` (cloud, dependency, docker, gcp, local, mysql). `sm-shop/src/main/resources/database.properties` is in `.gitignore` (line 30).
3. `shopizer-core-config.xml` has no `docker` profile, so `profiles/docker/database.properties` (H2, `./SALESMANAGER`, the demo database the image ships at `/SALESMANAGER.h2.db`) is not loaded by any profile.

The same jar starts when that file is put on the class path: in step 2 below it is taken out of the jar and given to Spring Boot's own `PropertiesLauncher` with `-Dloader.path`. The jar is not changed.

From the run below:

```
docker wait shopizer-as-published
1
FileNotFoundException: class path resource [database.properties] cannot be opened because it does not exist
unzip -l shopizer.jar | grep -o 'BOOT-INF/classes/.*database.properties'
BOOT-INF/classes/profiles/cloud/database.properties
BOOT-INF/classes/profiles/dependency/database.properties
BOOT-INF/classes/profiles/docker/database.properties
BOOT-INF/classes/profiles/gcp/database.properties
BOOT-INF/classes/profiles/local/database.properties
BOOT-INF/classes/profiles/mysql/database.properties
Started ShopApplication in 19.17 seconds
```

**What we are expecting**

The published image starts with its demo database when run as published.

**How to reproduce**

1. The image as published, and where `database.properties` is in its jar:

```bash
# Needs Docker and unzip.
docker run -d --name shopizer-as-published --platform linux/amd64 shopizerecomm/shopizer:3.2.7 >/dev/null
docker wait shopizer-as-published     # prints the exit code once the container has stopped by itself
docker logs shopizer-as-published 2>&1 | grep -m1 -o 'FileNotFoundException: class path resource \[database.properties\] cannot be opened because it does not exist'
docker rm shopizer-as-published >/dev/null

docker create --name shopizer-files --platform linux/amd64 shopizerecomm/shopizer:3.2.7 >/dev/null
docker cp shopizer-files:/opt/app/shopizer.jar ./shopizer.jar && docker rm shopizer-files >/dev/null
unzip -l shopizer.jar | grep -o 'BOOT-INF/classes/.*database.properties'
```

2. The same image and jar, with the jar's own `profiles/docker/database.properties` on the class path:

```bash
mkdir -p conf && unzip -p shopizer.jar BOOT-INF/classes/profiles/docker/database.properties > conf/database.properties
docker run -d --name shopizer --platform linux/amd64 -p 127.0.0.1:8080:8080 -v "$PWD/conf:/conf:ro" \
  shopizerecomm/shopizer:3.2.7 java -cp /opt/app/shopizer.jar -Dloader.path=/conf \
  -Dloader.main=com.salesmanager.shop.application.ShopApplication org.springframework.boot.loader.PropertiesLauncher >/dev/null
until curl -sf "http://127.0.0.1:8080/api/v1/store/DEFAULT?lang=en" >/dev/null; do sleep 3; done
docker logs shopizer 2>&1 | grep -m1 -o 'Started ShopApplication in [0-9.]* seconds'
```

**Possible fix**

Add a `docker` profile to `shopizer-core-config.xml` that loads `classpath:profiles/docker/database.properties` and set `SPRING_PROFILES_ACTIVE=docker` in the image; or put that file at the root of the class path when the image is built.
