---
title: Introducing helm-maven-plugin
layout: post
author: Paul Vorbach
date: 2019-04-02
tags:
  - helm
  - maven
---

Creating and maintaining Helm Charts for your services that run on Kubernetes
can be quite cumbersome. Typically building a Helm Chart involves the following
steps:

 1. Run `helm create`
 2. Tweak `template/deployment.yaml` and  `values.yaml` until you think it might
    be able to run
 3. Build your Chart using `helm package`
 4. (Optionally) Upload your chart to a Helm repository like
    [Chart Museum][chartmuseum]
 5. (Optionally) Test deployment using `helm upgrade` with `--dry-run`
 6. Deploy using `helm upgrade`

[chartmuseum]: https://chartmuseum.com/

Once anything in the process goes wrong, and it always does due to the nature of
Helm Charts essentially being just Go Templates, you go back to step 2 and try
again. This does not include building the service itself and the corresponding
Docker Image.

Of course this process should be automated in your build tool.
For Java projects, this usually means either Apache Maven or Gradle. Due to the
lack of good tooling for Java projects, we at Device Insight decided to build
our own Maven plugin, `helm-maven-plugin`, more than one year ago.

While the first versions only supported packaging your Helm Chart, uploading it
to a repository and only worked on Linux, newer versions added support for
multiple platforms and also introduced new goals for rendering (`helm template`)
and linting (`helm lint`).

Earlier this year, we eventually [released the plugin under the Apache License
2.0 on GitHub][repo].

[repo]: https://github.com/deviceinsight/helm-maven-plugin

It's quite easy to set up in your own project. Say, your project is called
`my-project`. Then you only need to include the following snippet in the
`<build><plugins></plugins></build>` section of your `pom.xml`.

```xml
<plugin>
    <groupId>com.deviceinsight.helm</groupId>
    <artifactId>helm-maven-plugin</artifactId>
    <version>2.1.0</version>
    <configuration>
        <chartName>my-project</chartName>
        <chartRepoUrl>https://helmcharts.example.com</chartRepoUrl>
        <helmVersion>2.13.1</helmVersion>
        <valuesFile>src/test/helm/my-project/values.yaml</valuesFile>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>package</goal>
                <goal>lint</goal>
                <goal>template</goal>
                <goal>deploy</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

If you don't configure the `chartName`, your module's `artifactId` is used by
default. Additionally, the `helm-maven-plugin` assumes your Helm Chart lives at
`src/main/helm/my-project` (i.e. `src/main/helm/my-project/Chart.yaml`,
`src/main/helm/my-project/templates/deployment.yaml` and so on). So you either
need to move your Helm Chart files to that location or specify a different
directory using `<chartFolder>path/to/my/chart</chartFolder>` in the
plugin `<configuration/>`.

Depending on your needs, you can remove the `lint`, `template` or `deploy`
goals, but `package` is a requirement for the other goals.

This is how a typical run of `mvn deploy` looks like, if this plugin is
configured:

```
[INFO] Scanning for projects...
[INFO]
[INFO] -----------------< com.example:my-project >------------------
[INFO] Building my-project 1.20.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- spring-boot-maven-plugin:2.1.3.RELEASE:build-info (build-info) @ my-project ---
[INFO]
[INFO] --- maven-resources-plugin:3.0.2:resources (default-resources) @ my-project ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource
[INFO] Copying 12 resources
[INFO]
[INFO] --- maven-compiler-plugin:3.7.0:compile (default-compile) @ my-project ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:3.0.2:testResources (default-testResources) @ my-project ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 6 resources
[INFO]
[INFO] --- maven-compiler-plugin:3.7.0:testCompile (default-testCompile) @ my-project ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-surefire-plugin:2.22.0:test (default-test) @ my-project ---
[WARNING] useSystemClassloader setting has no effect when not forking
[INFO] Surefire report directory: /home/pvo/workspace/my-project/target/surefire-reports
[INFO]
[INFO] [Surefire output skipped]
[INFO]
[INFO] Tests run: 25, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- maven-jar-plugin:3.0.2:jar (default-jar) @ my-project ---
[INFO] Building jar: /home/pvo/workspace/my-project/target/my-project-0.1.0-SNAPSHOT.jar
[INFO]
[INFO] --- spring-boot-maven-plugin:2.1.3.RELEASE:repackage (default) @ my-project ---
[INFO] Attaching archive: /home/pvo/workspace/my-project/target/my-project-0.1.0-SNAPSHOT-exec.jar, with classifier: exec
[INFO]
[INFO] --- docker-maven-plugin:0.28.0:build (docker-build) @ my-project ---
[INFO] Copying files to /home/pvo/workspace/my-project/target/docker/docker.example.com/my-company/my-project/latest/build/maven
[INFO] Building tar: /home/pvo/workspace/my-project/target/docker/docker.example.com/my-company/my-project/latest/tmp/docker-build.tar
[INFO] DOCKER> [docker.example.com/my-company/my-project:latest]: Created docker-build.tar in 303 milliseconds
[INFO] DOCKER> [docker.example.com/my-company/my-project:latest]: Built image sha256:05956
[INFO] DOCKER> [docker.example.com/my-company/my-project:latest]: Removed old image sha256:6fbc0
[INFO]
[INFO] --- helm-maven-plugin:2.1.0:package (default) @ my-project ---
[INFO] Clear target directory to ensure clean target package
[INFO] Created target helm directory
[INFO] Successfully packaged chart and saved it to: /home/pvo/workspace/my-project/target/helm/my-project-0.1.0-SNAPSHOT.tgz
[INFO]
[INFO] --- helm-maven-plugin:2.1.0:lint (default) @ my-project ---
[INFO] Output: ==> Linting my-project
[INFO] Output: [INFO] Chart.yaml: icon is recommended
[INFO] Output:
[INFO] Output: 1 chart(s) linted, no failures
[INFO]
[INFO] --- helm-maven-plugin:2.1.0:template (default) @ my-project ---
[INFO] Rendered helm template to /home/pvo/workspace/my-project/target/test-classes/helm.yaml
[INFO]
[INFO] --- maven-install-plugin:2.5.2:install (default-install) @ my-project ---
[INFO] Installing /home/pvo/workspace/my-project/target/my-project-0.1.0-SNAPSHOT.jar to /home/pvo/.m2/repository/com/example/my-company/my-project/0.1.0-SNAPSHOT/my-project-0.1.0-SNAPSHOT.jar
[INFO] Installing /home/pvo/workspace/my-project/pom.xml to /home/pvo/.m2/repository/com/example/my-company/my-project/0.1.0-SNAPSHOT/my-project-0.1.0-SNAPSHOT.pom
[INFO] Installing /home/pvo/workspace/my-project/target/my-project-0.1.0-SNAPSHOT-exec.jar to /home/pvo/.m2/repository/com/example/my-company/my-project/0.1.0-SNAPSHOT/my-project-0.1.0-SNAPSHOT-exec.jar
[INFO]
[INFO] --- maven-deploy-plugin:2.8.2:deploy (default-deploy) @ my-project ---
Downloading from snapshots: http://nexus.example.com/nexus/content/repositories/snapshots/com/example/my-company/my-project/0.1.0-SNAPSHOT/maven-metadata.xml
Uploading to snapshots: http://nexus.example.com/nexus/content/repositories/snapshots/com/example/my-company/my-project/0.1.0-SNAPSHOT/my-project-0.1.0-20190402.142507-6.jar
Uploading to snapshots: http://nexus.example.com/nexus/content/repositories/snapshots/com/example/my-company/my-project/0.1.0-SNAPSHOT/my-project-0.1.0-20190402.142507-6.pom
Downloading from snapshots: http://nexus.example.com/nexus/content/repositories/snapshots/com/example/my-company/my-project/maven-metadata.xml
Uploading to snapshots: http://nexus.example.com/nexus/content/repositories/snapshots/com/example/my-company/my-project/0.1.0-SNAPSHOT/maven-metadata.xml
Uploading to snapshots: http://nexus.example.com/nexus/content/repositories/snapshots/com/example/my-company/my-project/maven-metadata.xml
Uploading to snapshots: http://nexus.example.com/nexus/content/repositories/snapshots/com/example/my-company/my-project/0.1.0-SNAPSHOT/my-project-0.1.0-20190402.142507-6-exec.jar
Uploading to snapshots: http://nexus.example.com/nexus/content/repositories/snapshots/com/example/my-company/my-project/0.1.0-SNAPSHOT/maven-metadata.xml
[INFO]
[INFO] --- docker-maven-plugin:0.28.0:push (docker-push) @ my-project ---

[INFO] DOCKER> The push refers to repository [docker.example.com/my-company/my-project]
[INFO] DOCKER> latest: digest: sha256:e02917483ec8ae515c84211a591fe9dedbac17e01dd8adc71e027a8e9b2015f1 size: 2827

[INFO] DOCKER> Pushed docker.example.com/my-company/my-project:latest in 33 seconds
[INFO]
[INFO] --- helm-maven-plugin:2.1.0:deploy (default) @ my-project ---
[INFO] /home/pvo/workspace/my-project/target/helm/my-project-0.1.0-SNAPSHOT.tgz posted successfully
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:03 min
[INFO] Finished at: 2019-04-02T16:26:53+02:00
[INFO] ------------------------------------------------------------------------
```

Note that we are using the [`docker-maven-plugin`][dmp] here for building and
deploying Docker Images of our project. This is not done by `helm-maven-plugin`.

[dmp]: https://dmp.fabric8.io

As you can see above, `helm-maven-plugin` first packages the Chart into an
archive at `target/helm/my-project-0.1.0-SNAPSHOT.tgz`. Then it runs `helm lint`
using the `valuesFile` from the configuration (via the `-f` parameter). The output
is printed to the Maven log verbatim. After that, `helm template` renders the
Chart to `target/test-classes/helm.yaml` using the same values file. This way,
the rendered YAML file is available on the test classpath as
`classpath:helm.yaml`! You can write unit tests that validate the contents of
the rendered Helm Chart or check its contents by hand. As a last step, the
`deploy` goal uploads the packaged Chart to the configured `chartRepoUrl`.

We hope you find the plugin useful. It is available on Maven Central under the
following coordinates:

```xml
<groupId>com.deviceinsight.helm</groupId>
<artifactId>helm-maven-plugin</artifactId>
<version>2.1.0</version>
```

You can find the source code in a [GitHub repository][repo]. If you have any
problems or suggestions, feel free to open an issue there.

Happy Helming!
