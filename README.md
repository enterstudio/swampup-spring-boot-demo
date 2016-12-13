# Swampup Talk Script

> this code [lives on GitHub](https://github.com/jbaruch/swampup-spring-boot-demo)

## the Micro Microservice

* josh builds a REALLY micro microservice ("Hello World!") a simple spring boot rest app
* josh writes a simple unit test to test the REST endpoint
* josh makes it executable == true (reproducible deliverable software that requires only a JVM to run!)
* josh chucks it into github

##   Continuous Integration for Feedback

> TT: were not done! the whole point of a microservice is to support ease of iteration and agility. in order  to have that we need a fast feedback loop. We need continuous integration. We need to know that this software doesn't just work on my machine.

* baruch says we need continuous integration.
* we add a basic default `.travis.yml` to project and add repo on Travis.org

```
language: java
jdk: oraclejdk8
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
jdk: oraclejdk8
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

                            <contextUrl>https://cloudnativejava.jfrog.io/cloudnativejava</contextUrl>
                            <username>${env.ARTIFACTORY_USERNAME}</username>
                            <password>${env.ARTIFACTORY_API_KEY}</password>
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
* make sure local environment *AND* Travis have `ARTIFACTORY_USERNAME` and `ARTIFACTORY_API_KEY`

here is an updated travis.yml with the valid, encrypted values as derived by saying: `travis encrypt ARTIFACTORY_API_KEY=... --add` and `travis encrypt CF_PASSWORD=... --add` while in the directory containing our .travis.yml file.

```
language: java
jdk: oraclejdk8
script: "./mvnw deploy -Dbuild.number=${TRAVIS_BUILD_NUMBER}"
install: echo installing
env:
  global:

  #  Artifactory (ARTIFACTORY_USERNAME && ARTIFACTORY_API_KEY)
  - ARTIFACTORY_USERNAME=admin
  - secure: W/HnrxsDRyG66QyRRBtLSgkCnL7DLMN5o8HQuJQCbEvCgh/X/SzHSW3f8vMXiRN4aECEvyn6OYN84EAb/5/BXC7DqqZX0nnvrQtHHZizaL5JfGgH0rF1rddr4RycZhFlcxZz/Yk1y0XQyRQdqfm9pv4FkoCUcpXgZiXYbuioA6KXuC1fHb86M8kF1q+DPkfktB/j1A3ir8h8vxTKRmDM4hLDwPY5mzCZ369QksVo4vTYwuLxXaOG0YiBWhTyqo/KsVsso92rKKnJ6Dauk1MjKVgndemaG8ivVMSale0RdSpfnR4hxkE3HKWGrkM1YmF8MvvOvMfzO0FoHhkr1dsGM1LaJhrdh5VD0h3W0Bf+xQvPx4uQqpAX6o/aoAXUmKpJxYIhJJH0/ZhAcvRVZ2SaMesvueuJSV3uxxEJZ01PnQgOuGDo94vtQ+sGh2xOPxNLA/E6lBizMKSGURGvaBHXrXPT4VWNT4a14CVvMPA3D1Y9gtXya5NY1eqPs//LnUSAN9c+mgEM9ghfoHe/BRkbADfDKokGlmiHofsluSHy+mTC0qaE2UaDxhqdAbVqGON1Fp4/sVHJ13N7P8vf7EpnGlmAuecL8+CpZlTHBQsnvHOaaeLl7W5WXRiT3DpvMclIZEvYt6+n/JjhWtxjanP71+36pI5286vRk8zDLgRitKg=

  #  CloudFoundry
  - CF_API=http://api.run.pivotal.io
  - CF_SPACE=joshlong
  - CF_ORG=platform-eng
  - CF_USER=starbuxman@gmail.com
  - secure: kLZ6mWu79U39hlZyiMYwe1vj7asLJ/s57AjH5Hanorq8e4biXe1Mx6ZTZRRXDRizmps6liOJFsZLgIss/SECf3uiNBNb034Df7Js2dUPWBohZJOV6rA5IaHfgQVPg93oCBvczB3uqTammLs+c9yuQ/75npQtUAtHz+hrLm37bTCprr2UHgucoDPVfMnP+f2XfcEnAxYtEZpmxiZuudhCLDoyyfA2IjHph93pNm6tAJ+XF9SgHTewWXjkr+USmT1cszniykOBSaib4ymwVrn1MjuGS3HbK4W6tRsFtAumRNoX1PM2LKjP9u+hl4ZM0CZ7fb1dCa8fzqiYFiORS+LyUshhd03tB92ukdypA6RvUmMFXLtWhg8Eh77k5B20mHoWvBJaQOIHF+ppPuVodPlbHkzR44BHl8eDK/MKSJK/aN1x/ZQNrEngOfUy3ToWpgO+neyTmkSUZ6yWDamiovo4Gy7Ny+SaQ926kOZjdRLIsGzgAsBbQdeRbGUdP7Qh6bbnAjYB3TcfB9U3b9mZNTJ9Vr0ivwHqmgElUjP9AH6xAILDybQ9gBoHlH4gObC2es2ogpkpPYVCiMLI3bHFoHnTp4ExVXFC2XMzk1T67lshHNzPVWct3EH9ONxWwHtRWUmpHaFnynLQqZHqQPjjyvnJh/KrDq3IlzfVmk/9unBZ/Pk=
```

> TT: Travis doesn't let us override the default Maven repository settings.  Luckily, start.spring.io gives us a Maven wrapper setup that we can tweak to use our own Maven distribution and give us reproducible builds. This maven distribution will be aware of our Artifactory repository (which in turn knows about JCenter, Maven central, company repositories, etc). We can use Maven wrapper to download a customized Maven distribution that has a `.settings.xml` that knows about my custom Artifactory. We support this goal by DL'ing the Maven distribution listed in the Maven wrapper's `wrapper.properties`, unpacking it, changing the `settings.xml` that's within to point to our Artifactory (We were able to generate those settings by going to any Maven repository, for example 'libs-release-local', clicking "Set Me Up", and then choosing "Maven" as the tool, and selecting Generate Maven Settings. It'll prompt you to confirm the setup. Click "Generate Settings". Then copy resulting settings to clipboard and paste in $MAVEN_DISTRO/conf/settings.xml.), then deploying that re-packaged .zip distribution to our artifactory instance (we logged into Artifactory, went to artifactory repository browser, then clicked on "distributions"- the repositorty for our custom Maven distribution. Then, clicked on Deploy and uploaded a Maven distributions that's been customized to have the correct Maven settings), then changing the `distributionURL` to point to our artifactory.

```

distributionUrl=https://cloudnativejava.jfrog.io/cloudnativejava/distributions/apache-maven-3.3.3-bin.zip
```


* now `travis.yml` is:

```
language: java
jdk:
 - oraclejdk8
script: ./mvnw deploy -Dbuild.number=${TRAVIS_BUILD_NUMBER}
install: echo installing
```

## Supporting QA and Smoke Tests

* this will successfully run tests, deploy to artifactory. what about staging and QA? and smoke tests? now we need to *also* publish to CF so somebody can then review it in QA/staging and - if it's good - promote it. let's modify travis.yml:

```
language: java
script: ./mvnw deploy -Dbuild.number=${TRAVIS_BUILD_NUMBER} && ./bin/cf.sh
install: echo 'installing'
jdk:
  - oraclejdk8
```


## To Production!

* Somebody in QA reviews the change, accepts it (perhaps in Pivotal Tracker!), then PT should somehow call the following REST call to promote the build from `libs-staging-local` to `libs-release-local`

```
http -a admin POST https://cloudnativejava.artifactoryonline.com/cloudnativejava/api/build/promote/demo/9 status=promoted comment="let's promote it" targetRepo=libs-release-local
```



* once it's in release repository on artifactory, let's (also release to bintray, though this might be optional with distribution repositories) AND publish from bintray. We want to make sure that when we release to bintray that there is a webhook in place to automatically publish to CF and do blue/green builds.

* configure a release repository on artifactory. goto admin -> distribution -> add a new distribution repository. then, authenticate with bintray.

> TT: why do we need another repisitory like bintrary if artifactory is a repository and we already have the artifacts are in the release repository? Well, you COULD. But there are some drawbacks. On-Prem Artifactory isn't meant to handle the load of binary distributation. It doesnt have knowledge of CDNs, etc. Also, short of harvesting the logs, you don't have a builtin single pane of glass experience showing you downlaods and stats about the binaries. Enter bintray, a hosted distribution hub.

Now, let's release to bintray and publish release to the world (Hopefully, we'll be able to skip this first `http` call because JFrog will have announced distribution repositories)

```
http -a admin POST https://cloudnativejava.artifactoryonline.com/cloudnativejava/api/build/pushToBintray/demo/9 subject=swampup-cloud-native-java repoName=maven packageName=demo versionName=9

http -a joshlong POST https://api.bintray.com/content/swampup-cloud-native-java/maven/demo/9/publish
```

Now, we also need to release in CF. It is done by blue/green CF switch. Do we have a REST API for the switch? We don't, but we could! Josh is writing the app, and it's AMAZING. Now we have a REST API to call for the switch flip. We call it from the Bintray Deployment web hook.


```
http -a joshlong POST https://api.bintray.com/webhooks/swampup-cloud-native-java/maven/demo/ \
    url=$MY_CUSTOM_BINTRAY_WEBHOOK_WHICH_WILL_BE_A_CF_APP_THAT_DOES_BLUE_GREEN_DEPLOY_OF_APP \
```

Now when new version is released on Bintray (which means we depoyed!) we flip the switch and it goes live on CF as well.
