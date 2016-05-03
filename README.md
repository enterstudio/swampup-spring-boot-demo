# Swampup Talk Script

> this code [lives on GitHub](https://github.com/jbaruch/swampup-spring-boot-demo)

* josh builds a REALLY micro microservice ("Hello World!") a simple spring boot rest app
* josh writes a simple unit test to test the REST endpoint
* josh makes it executable == true (reproducible deliverable software that requires only a JVM to run!)
* josh chucks it into github

## TT: were not done! the whole point of a microservice is to support ease of iteration and agility. in order  to have that we need a fast feedback loop. We need continuous integration. We need to know that this software doesn't just work on my machine.

* baruch says we need continuous integration.
* we add a basic default `.travis.yml` to project and add repo on Travis.org
```
language: java
jdk:
 - oraclejdk8
```
> TT: this runs CI but its not good enough! we're throwing away the artifact and we're not building in a window for staging. This is Continuous Delivery/Deployment: we want to potentially release to production on *every* git push!! this means that every build is potentially releasable. This means we have to change the way we think about versioning. So far we've been using Mavens naive notion of `SNAPSHOTs`, but this no longer applies. Every build that makes it into the promoted repository on Artifactory could be a released build. We also don't want to invite the nightmare of Maven release plugin since we don't need to have a formal maven step for release. Well let the process do it. But we can't release a non-final version at the end of the line, either! So, Let's revisit the build and change where it derives its version number:

In Maven it becomes:
```
<version>${build.number}</version>
```
and then we specify a default value for this property in the Maven build:
```
<build.number>0.0.1-SNAPSHOT</build.number>
```

When we run the code on Travis, though, we'll override the build version with the following incantation which will use a Travis environment variable, `TRAVIS_BUILD_NUMBER`:

```
language: java
jdk:
 - oraclejdk8
script: ./mvnw deploy -Dbuild.number=${TRAVIS_BUILD_NUMBER}
install: echo installing

```

* we need to push the artifact to artifactory
* add Artifactory Maven plugin to Maven Build and add environment variables for authentication. This wil be configured on local machines and in Travis
```
        <plugin>
            <groupId>org.jfrog.buildinfo</groupId>
            <artifactId>artifactory-maven-plugin</artifactId>
            <version>2.4.0</version>
            <inherited>false</inherited>
            <executions>
                <execution>
                    <id>build-info</id>
                    <goals>
                        <goal>publish</goal>
                    </goals>
                    <configuration>
                        <deployProperties>
                            <build.vcsRevision>{{TRAVIS_COMMIT}}</build.vcsRevision>
                        </deployProperties>
                        <publisher>
                            <contextUrl>https://cloudnativejava.artifactoryonline.com/cloudnativejava</contextUrl>
                            <username>${env.ARTIFACTORY_USERNAME}</username>
                            <password>${env.ARTIFACTORY_PASSWORD}</password>
                            <repoKey>libs-staging-local</repoKey>
                            <snapshotRepoKey>libs-snapshot-local</snapshotRepoKey>
                        </publisher>
                        <buildInfo>
                            <agentName>Travis CI</agentName>
                            <buildNumber>{{TRAVIS_BUILD_NUMBER}}</buildNumber>
                            <buildUrl>http://travis-ci.org/{{TRAVIS_REPO_SLUG}}/builds/{{TRAVIS_BUILD_ID}}
                            </buildUrl>
                            <principal>{{USER}}</principal>
                            <vcsRevision>{{TRAVIS_COMMIT}}</vcsRevision>
                        </buildInfo>
                    </configuration>
                </execution>
            </executions>
        </plugin>

```
* make sure local environment *AND* Travis have `ARTIFACTORY_USERNAME` and `ARTIFACTORY_PASSWORD`

** PROBLEM: Travis doesn't let us override the default Maven repository settings.  Luckily, start.spring.io gives us a Maven wrapper setup that we can tweak to use our own Maven distribution and give us reproducible builds. This maven distribution will be aware of our Artifactory repository (which in turn knows about JCenter, Maven central, company repositories, etc). We can use Maven wrapper to download a customized Maven distribution that has a `.settings.xml` that knows about my custom Artifactory. We support this goal by DL'ing the Maven distribution listed in the Maven wrapper's `wrapper.properties`, unpacking it, changing the `settings.xml` that's within to point to our Artifactory, then deploying that re-packaged .zip distribution to our artifactory instance, then changing the `distributionURL` to point to our artifactory.

```
distributionUrl=https://cloudnativejava.artifactoryonline.com/cloudnativejava/distributions/apache-maven-3.3.3-bin.zip
```

We logged into Artifactory, added a new repository called "distributions", then clicked 'Deploy' button. Go to the deployed distribution, copy the 'Download' link and then point wrapper.settings to that.

* now `travis.yml` is:

```
language: java
jdk:
 - oraclejdk8
script: ./mvnw deploy -Dbuild.number=${TRAVIS_BUILD_NUMBER}
install: echo installing
```

* this will successfully run tests, deploy to artifactory.

* what about staging and QA? and smoke tests? now we need to *also* publish to CF so somebody can then review it in QA/staging and - if it's good - promote it. let's modify travis.yml:

```
language: java
script: ./mvnw deploy -Dbuild.number=${TRAVIS_BUILD_NUMBER} && ./bin/cf.sh
install: echo 'installing'
jdk:
  - oraclejdk8
```


# BIG GREEN BUTTON TERRITORY

* Somebody in QA reviews the change, accepts it (perhaps in Pivotal Tracker!), then PT should somehow call the following REST call to promote the build from `libs-staging-local` to `libs-release-local`

```
http -a admin POST https://cloudnativejava.artifactoryonline.com/cloudnativejava/api/build/promote/demo/9 status=promoted comment="let's promote it" targetRepo=libs-release-local
```



* once it's in release repository on artifactory, let's (also release to bintray, though this might be optional with distribution repositories) AND publish from bintray. We want to make sure that when we release to bintray that there is a webhook in place to automatically publish to CF and do blue/green builds.


> TT: why do we need another repisitory like bintrary if artifactory is a repository and we already have the artifacts are in the release repository? Well, you COULD. But there are some drawbacks. On-Prem Artifactory isn't meant to handle the load of binary distributation. It doesnt have knowledge of CDNs, etc. Also, short of harvesting the logs, you don't have a builtin single pane of glass experience showing you downlaods and stats about the binaries. Enter bintray, a hosted distribution hub.

Now, let's release to bintray and publish release to the world (Hopefully, we'll be able to skip this first `http` call because JFrog will have announced distribution repositories)

```
http -a admin POST https://cloudnativejava.artifactoryonline.com/cloudnativejava/api/build/pushToBintray/demo/9 subject=swampup-cloud-native-java repoName=maven packageName=demo versionName=9

http -a joshlong POST https://api.bintray.com/content/swampup-cloud-native-java/maven/demo/9/publish
```

We can even go further, and specify a webhook on release publishing that in turn cna do whatever we want, including blue/green CF deployments, like this:

```
http -a joshlong POST https://api.bintray.com/webhooks/swampup-cloud-native-java/maven/demo/ \
    url=$MY_CUSTOM_BINTRAY_WEBHOOK_WHICH_WILL_BE_A_CF_APP_THAT_DOES_BLUE_GREEN_DEPLOY_OF_APP \
    method=post

```
