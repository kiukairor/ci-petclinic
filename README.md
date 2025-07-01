# In Scope and Out of Scope
Based on the topic, we focused on the Continuous Integration side of things.  
We relied on https://github.com/spring-projects/spring-petclinic, removed the gradle settings and treated the underlying maven project as the main starting point of our pipeline.
Hence, folder src/ has been left unmodified.

We have considered branch management and Continous Deployment slightly out of scope.
We made the (somewhat arbitrary) choice to disregard GitHub branch management. As an easy shortcut to enable minimal branch protection, we could have enabled branch protection to prevent pushing directly to the main branch and go through Pull requests. But we kept disabled to trigger the CI pipeline more easily.

This has lead to a few unnatural choices in the pipeline design, that are explained inline (e.g. maven release in *00_scan_build_publish.yaml*)

Note that the different repos matching different environments are quite limited, but we allowed artifact promotion at some point in the pipeline to leverage JFrog promotion mechanisms.

# What was done
1. Use spring-petclinic as the starting project for the pipeline
2. Build a CI pipeline, mainly relying on GitHub Actions and JFrog Artifactory and Xray for the security side of things
3. When the CI pipeline is triggered (on push event)
    a. 00_scan_build_publish_jar.yaml run quality tests (checkstyle, spotbugs, jacoco) and security scans (XRay via `jf scan`) on the built artifact before uploading. If builds and tests pass, it uploads the SNAPSHOT Jar to the maven Snapshot repo.
    b. We then ran a bit more tests (maven verify and scan the artifact after upload to ensure it passes the centralized security watch and policies). Upon validation, we decided to rebuild this maven project in non SNAPSHOT mode and upload it to our maven dev repo.
    c. 01_download_containerize.yaml runs after 00_scan_build_publish_jar.yaml completes and download the dev jar. It then uses it to build a Docker image. The Docker image is scanned and then uploaded to Artifactory.
    d. 02_test_and_promote.yaml fetches this docker image from our Docker Artifactory repo and an helm chart is then leveraged to build a K8s application out of the container image. Through GitHub Actions, this container is deployed to a kind cluster for (very) basic functionality testing. Upon success, we made the optimistic assumption that the different artifacts were ready to be promoted. Because we only had a maven release local repo, this promoted the dev jar to the release repo. Same logic should apply to container image and helm chart.

4. & 5. Along the way we have tried to follow some of the industry good practises:
- As already mentioned we did not strictly follow Dev artifacts (snapshot, dev) released into QA, Pre-Prod, Prod releases but we did perform some minimal isolation for the sake of it.
- Resolving dependencies (e.g. Maven, Docker) does not occur directly form public repos (Maven Central, DockerHub) but these dependencies were proxied through Artifactory respective repos.
- In the same spirit, Artifactory repos consisted in Virtual Repos made of local repos for deploying local artifacts and remote repos to fetch public dependencies. This allow caching, and granular control in case some public repos need to be blocked.
- As much as possible we have tried to avoid using plain access tokens (exception of the need for maven lifecyles, see explanation in *00_scan_build_publish_jar.yaml*) and preferred the use of JWT token through the integration between Git Hub Actions and Artifactory builds.
- For promoting artifacts (02_test_and_promote.yaml), we followed recommended patterns (jf rt build-promote) rather than copying artifacts from one place to the other.
- We minimised the use of flag 'latest' in the overall pipeline.

6. We introduced/used some initial quality and security checks from workflow 00_scan_build_publish_jar.yaml, where the idea is to come up with loose conditions that could block the initial build and the full pipeline if too many unexpected results were found (shift left paradigm). These tests are light and should enable the pipeline to continue.
Build (especially Docker images) were tagged with the commits numbers for traceability. All the artifactory builds and uploads were also traced thanks to, for example, commands like 'jf rt build-collect-env' and 'jf rt' build-add-git'. 
Optionally, the pipeline is designed in a way that it could fail if the artifacts being manipulated have not been created at the back of the of the same commit.

By default Artifactory builds SBOM of uploaded artifacts (Xray > Scans List > * > SBOMs). And the spring-petclinic project comes with the plugin cyclonedx-maven-plugin whill allows to build enriched SBOM using jf cli.

We also enabled security watches tied to security policy to notify/faild builds/block download/... depending on the security policies and corresponding violations.
It allows to quarantine vulnerables artifacts. As an illustration, we have made some strict policy on the release maven repo which triggered a violation. Relying on this policy, we blocked the distribution of this *vulnerable* artifact.

## What is missing, what to improve.

We did not introduce branch management and advanced logic around it so the corresponding logic for different repos and environments might be missing at the Artifactory level.
Dockerfile and Helm charts are quite minimal and did not undergo optimsation and advanced security consideration:
 - The docker image was not layered and appears to be quite big
 - The helm chart is not ready for production deployment (only one replica, no security context), it is mainly there to prove the container image can be deployed into a cluster
 
Above all, we have not introduced signatures of the different artifacts within the pipeline to ensure integrity **and** authenticity of the binaries.
Note that when downloading artifacts, Artifactory is meant to verify integrity (and security watches) before giving access to the requested binaries.


Some bad practises have also been followed in this project: some over-privileged tokens or accounts may have been used. 
Repo names and many other things are hardcoded in the workflow files, which makes it a bit unconvenient.

# Howto

## Pre-reqs and Env Variables

For this project to work as expected, you will need relevant repos on Artifactory:

- Create a trial account on Jfrog to use Artifactory and Xray features
- Create a maven virtual Snapshot repo *my-repo-libs-snapshot* made of a local repo *my-repo-libs-snapshot-local* (defaulted for deployment) and a remote one *my-repo-maven-remote*
- Create a maven virtual Dev repo *my-repo-libs-dev-virtual* made of a local repo *my-repo-libs-dev-local* (defaulted for deployment) and the remote repo *my-repo-maven-remote*
- Create a maven virtaul Release repo *my-repo-libs-release* made of a local repo *my-repo-libs-release-local* (defaulted for promotion) and *my-repo-maven-remote*
- Create a docker virtual repo *my-docker-repo-docker* made of *my-docker-repo-docker-local* and *my-docker-repo-docker-remote*
- Additonaly, create a helm repo (not used currently)
These repos have been configured following the Artifcatory wizard that allows to *Set up Client/ Tools* to generate tokens that may be needed in the pipeline workflow. In partiuclar, while setting the Maven client repos, the wizard generated some username/password that we used as environment variables in GitHub Actions (ARTIFACTORY_USERNAME and ARTIFACTORY_PASSWORD). 

The pipeline is also leveraging OIDC integration (Provider GitHub) with Audience set to *my-aud* and 3 Identity mappings with the following JSON claims (each mapping for each workflow in the pipeline) :
{"workflow": "Scan, Build and Publish Jar"}, {"workflow": "Download and Containerize"} and {"workflow": "Helm Chart and Promote artifacts"}
This OIDC integration allows us to use JF CLI tool in the pipeline without the use of long-lived access token

## Howto
- Git clone this project. 
- Make a dummy change in the first workflow file or in the source code (folder src/); 
- Commit and push
- Observe the CI pipeline going into actions.

**Note** We acknowledge real life scenario would benefit a realbranch management and such workflow might be initiated by approved pull requests, on dev branches for example.

# Docker image command
Upon download and extraction of the docker image, 
One would simply run: 
`docker run -p 8080:8080 trial3sr477.jfrog.io/my-docker-repo-docker/spring-petclinic:latest`  
And access the service by browsing to *http://\<docker-machine-ip\>:8080*.

