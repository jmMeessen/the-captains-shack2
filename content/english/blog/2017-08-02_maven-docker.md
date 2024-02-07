+++
date = "2017-08-02T17:46:17+02:00"
draft = false
title = "Maven and Docker, some thoughts..."
author = ""
comments = true
image = "images/blog/old/mvn-docker.png"
menu = ""
share = ""
slug = "MavenAndDocker"
tags = [ "docker", "maven", "java" ]
categories = ["docker" , "development"]
+++


***

**[TL;DR]**

Various strategies are available when using Docker-based Integration test in the Java development lifecycle. This article describes and compares a Maven-centric strategy and a more loosely coupled one.

***

Integration tests with Docker is a dream come true. 

For years, I have been struggling to get hold of Test Environments where I could validate my software. But now Docker makes it easy and cheap. Further, it allows validation on near Production configuration. 
Docker is also a powerful tool for efficient cooperation between developers and production. OPS people can actively participate in the design and configuration of the deployed application.

## The Maven Docker Plugin

As soon as I learned about Docker, I actively used it in my Java development. It allowed me to automatically test cases not easily tested with large databases shared with multiple developers. On my own Oracle Database container, I could test complex database structure alteration (and roll back) for example. [Flyway](https://flywaydb.org/) and [DBunit](http://dbunit.sourceforge.net/) are fundamental tools to achieve efficient database test automation. 

**Flyway** is a handy database versioning tool. **DBunit** allows to setup tables and data before running tests and validate that the resulting data is the expected one.

Maven is now the mainstream Java build tool. Beside efficient dependency management, it combines many useful tools for managing unit tests, code quality checks or integration tests.  For a Java developer, integrating Docker "à la Maven" comes naturally. The plugin I felt most comfortable with is [fabric8io/docker-maven-plugin](https://dmp.fabric8.io/). It is fairly complete and regularly maintained.

This plugin allows to execute the equivalent Docker commands (build, run, stop, etc) as Maven Goals and attach them to phases. What is usually done is to attach the `docker:build` and `docker:start` goals to the `pre-integration-test` phase and the `docker:stop` to the `post-integration-test` phase. In the example below, an Oracle database is started and configured before the actual integration tests are run (java test classes ending with IT). When the integration tests complete, the docker container is stopped. Note that the example below uses fixed ports and addresses. It is not Continuous Integration ready but can easily be fixed. But this is another story.

{{< highlight xml >}}
<profile>
    <id>withIntegrationTest</id>
    <!--These tests can come in the way of somebody that doesn't have a docker ready environment. So it is better to give a way to skip them-->
    <build>
        <plugins>

            <!-- Integration test plugin (using defaults)-->       
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>${failsafe.maven.plugin.version}</version>
                <executions>
                    <execution>
                        <id>integration-test</id>
                        <goals>
                            <goal>integration-test</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>verify</id>
                        <goals>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            
            <!-- Docker related stuff -->
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>${docker.maven.plugin.version}</version>
                <configuration>
                    <logDate>default</logDate>
                    <images>
                        <!-- Docker Image to use -->
                        <image>
                            <alias>ODS_DB</alias>
                            <!-- Docker Image-->
                            <name>alexeiled/docker-oracle-xe-11g:latest</name>
                            <run>
                                <ports>
                                    <port>49160:22</port>
                                    <port>49161:1521</port>
                                    <port>49162:8080</port>
                                </ports>
                                <wait>
                                    <log>DB server startup</log>
                                    <time>30000</time>
                                </wait>
                            </run>
                        </image>
                    </images>
                </configuration>
                <executions>
                    <execution>
                        <id>start</id>
                        <phase>pre-integration-test</phase>
                        <goals>
                            <goal>build</goal>
                            <goal>start</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>stop</id>
                        <phase>post-integration-test</phase>
                        <goals>
                            <goal>stop</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- Daatabase setup stuff -->
            <plugin>
                <groupId>org.flywaydb</groupId>
                <artifactId>flyway-maven-plugin</artifactId>
                <version>${flyway.maven.plugin.version}</version>
                <configuration>
                    <url>jdbc:oracle:thin:@//localhost:49161/xe</url>
                    <user>system</user>
                    <password>oracle</password>
                    <schemas>
                        <schema>FLYWAY</schema>
                    </schemas>
                    <baselineOnMigrate>true</baselineOnMigrate>
                </configuration>
                <executions>
                    <execution>
                        <id>dbSetup</id>
                        <phase>pre-integration-test</phase>
                        <goals>
                            <goal>migrate</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            
        </plugins>
    </build>
</profile> 
{{< /highlight >}}

This setup is very natural for someone used to Maven and familiar with its XML syntax. It easily answers the two main use cases: 

* run all tests (unit and integration) automatically with a single `mvn verify` or `mvn install` command.
* start and setup the integration environment for debugging with an IDE (`mvn docker:start`) and then tear it down (`mvn docker:stop`).

But I came to the conclusion that for more complex cases, this was leading to a dead end.

### Strong points

1. If one is "Maven literate", its use is natural. No need to be a Docker expert.
2. It is easy to integrate into existing build (Maven) workflows.
3. A single command will setup the Docker environment needed for interactive Integration tests.

### Weak points

1. Adds an extra layer on top of the Docker API. It needs to be adapted at every new release (and we all know that the project is very prolific).
2. The plugin is not suited for handling complex infrastructure (as described in docker-compose files). It is tedious, verbose and especially in a "language" that is not well practiced outside of the Java Developer world. Note that there has been some effort to support docker-compose syntax as a descriptor for the plugin. But it keeps running behind the real product as it is a re-write of the original tool.
3. As a consequence of the above point, the plugin misses an important aspect of the Docker technology: being a DevOps enabler.

### Not using Docker at its best

Docker is a catalyst for improved cooperation between the development and the infrastructure/operation world. While Devs can enter the world of infrastructure and Ops can participate early in the development cycle, each pole has its tools of the trade. Docker-compose files are the tools of the trade to describe complex Docker infrastructure (container dependencies, networks, volumes, etc.)

With the Maven plugin specific language, we create a communication gap between the two entities. The goal should be that the Ops project contributor supplies the infrastructure part in such a way that it can be used from the early stages of Dev continuously up to the Production. This is particularly true for Integration tests where the more Production-like the setup the better the quality of the testing.

Thus, the docker-compose file and tool should best be used end to end in the application lifecycle.

*The Docker Maven Plugin is best suited for simple integration tests (single container like a database as in the sample) and when continuous delivery is not the goal.* 

## Integrating Docker-compose in Maven

In one of my side projects (adding integration tests to the Jenkins Gogs WebHook plugin), I was confronted with a more complex test infrastructure. I needed to start a Gogs Git server and make it talk with a Jenkins Server. It also involved several data volumes. An additional (but classical) complexity was to load the artifact under test in the Jenkins container and a git project to load in the Gogs GIT server.

The solution experimented in this project was based on the `exec-maven-plugin`. It executes a bash file to build/start and to stop the test environment described in a Docker-compose file. This allows to sub-contract the design and maintenance of the infrastructure to a team member that is not Maven literate. The Docker-compose file could also be used for further stages of a deployment.

This solution achieved the goals that I had set:

* using the Docker and Docker-compose API to its full extent and in its native form.
* integrate seamlessly in the Maven build lifecycle
* start and tear down the test environment with single commands.

This is how I proceeded.

Note: a demonstration of this setup can be seen in the [jenkinsci/gogs-webhook-plugin](https://github.com/jenkinsci/gogs-webhook-plugin) project. A pull request with this is Work In Progress at the time of writing.

Under the `src/test` folder, a `docker` subdirectory is used to hold the Docker-related data (Dockerfiles, compose files, etc). The `script` sub-directory holds the startup and teardown scripts (`scripts`). It is in the same folder that the reference git projects are stored in the `test-repos` sub-directory.  

With the `maven-resources-plugin`, the newly built artifact (the compiled plugin candidate) is pushed to the docker directory. From there, it can be included in the Jenkins container.

The same plugin is used to copy the reference demo Git repositories. They are copied to the `target` directory. The Integration Test program will use that data to set up the Git server.

The `exec-maven-plugin` is used to trigger the `start-test-environment.sh` script. It first launches a build of the images defined in the compose file. I chose to force the download of the upstream images at every build. Once all containers built, the configuration described in the docker-compose file is started.

{{< highlight bash >}}
#!/usr/bin/env bash

docker-compose -f src/test/docker/docker-compose.yml build --no-cache --pull
docker-compose -f src/test/docker/docker-compose.yml up --force-recreate -d
{{< /highlight >}}

During the `post-integration-test` phase, the `exec-maven-plugin` is used to trigger the `stop-test-environment.sh` that with tear down the docker-compose environment. It stops the infrastructure and removes all the volumes and dead containers.

{{< highlight bash >}}
#!/usr/bin/env bash

docker-compose -f src/test/docker/docker-compose.yml stop
docker-compose -f src/test/docker/docker-compose.yml rm -v --force
{{< /highlight >}}

This is how this is orchestrated with Maven.

{{< highlight xml >}}
<profile>
    <id>withIntegrationTest</id>
    <!--These tests can come in the way of somebody that doesn't have a docker ready environment. So it is better
    to give a way to skip them-->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>${failsafe.maven.plugin.version}</version>
                <executions>
                    <execution>
                        <id>integration-test</id>
                        <goals>
                            <goal>integration-test</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>verify</id>
                        <goals>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <artifactId>maven-resources-plugin</artifactId>
                <version>${resources.maven.plugin.version}</version>
                <executions>
                    <execution>
                        <id>copy-HPI-to-dockerDir</id>
                        <phase>pre-integration-test</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${basedir}/src/test/docker/jenkins-dockerfile</outputDirectory>
                            <resources>
                                <resource>
                                    <directory>${basedir}/target</directory>
                                    <includes>
                                        <include>gogs-webhook.hpi</include>
                                    </includes>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                    <execution>
                        <id>copy-testSources</id>
                        <phase>pre-integration-test</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <resources>
                                <resource>
                                    <directory>${basedir}/src/test/test-repos</directory>
                                    <includes>
                                        <include>**/*</include>
                                    </includes>
                                </resource>
                            </resources>
                            <outputDirectory>${basedir}/target/test-repos</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>${exec.maven.plugin.version}</version>
                <executions>
                    <execution>
                        <id>start-test-environmebt</id>
                        <phase>pre-integration-test</phase>
                        <goals>
                            <goal>exec</goal>
                        </goals>
                        <configuration>
                            <executable>sh</executable>
                            <arguments>
                                <argument>${basedir}/src/test/scripts/start-test-environment.sh</argument>
                            </arguments>
                        </configuration>
                    </execution>
                    <execution>
                        <id>stop-test-environment</id>
                        <phase>post-integration-test</phase>
                        <goals>
                            <goal>exec</goal>
                        </goals>
                        <configuration>
                            <executable>sh</executable>
                            <arguments>
                                <argument>${basedir}/src/test/scripts/stop-test-environment.sh</argument>
                            </arguments>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

        </plugins>
    </build>
</profile>
{{< /highlight >}}

## Conclusion

With this, I can use complex test configuration while keeping it easily readable in standard formats (Docker-compose). The environment definition is portable across environment (Dev, Test, Prod).

The next step is to improve the portability further so that these tests can be run in a complex environment like a Jenkins Integration Server (no fixed ports for example).

But this is another story....