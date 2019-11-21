# CICD for container images



- [default behaviour](#default-behaviour)
- [available parameters](#available-parameters)
- [Testing](#testing)
- [Ansible Tower](#ansible-tower)
- [Examples](#examples)
- [push registry settings](#push-registry-settings)
- [more complex Jenkinsfile](#more-complex-jenkinsfile)



This CI component will build and optionally test one or multiple Docker images.

**minimal Docker build Jenkinsfile**

```groovy
buildDockerImage([imageName: "myImage"])
```

## default behaviour

Assuming we are building with imageName "myImage"

The Dockerfile in the repository root will be built (with the repo root as build context), the equivalent of

**build image**

```shell
docker build .
```

On the *branch "develop"* the image will be pushed like 

**push branch**

```shell
docker tag <image_id> myImage:develop 
docker push myImage:develop docker.artifactory.imp.ac.at/it/myImage:develop
```

When building a git tag *v1.2.3*



**push tag**

```shell
docker tag  myImage:v1.2.3
docker push myImage:v1.2.3 docker.artifactory.imp.ac.at/it/myImage:v1.2.3
docker push myImage:v1.2.3 docker.artifactory.imp.ac.at/it/myImage:latest
```

i.e.

- builds from branch develop (or defined in parameter "pushBranches") will be pushed using the name of the branch as docker tag (never "latest").
- builds form tags will use the git tag also as the docker tag. additionally the docker image will be pushed tagged as "latest"



all pushed images will contain multiple metadata labels with information about the Jenkins build and the git repo information that was used for the build, see the vbc.* labels below

**docker inspect image: vbc.\* labels**

```json
`$ docker inspect docker.artifactory.imp.ac.at/it/orgt:develop | jq .[0].Config.Labels``{`` ``"architecture": "x86_64",`` ``"authoritative-source-url": "registry.access.redhat.com",``[snip]`` ``"vbc.git.branch": "develop",`` ``"vbc.git.commit": "53d90a63f4a54403e2d1508137c06c503d765f05",`` ``"vbc.git.tag": "false",`` ``"vbc.git.url": "ssh://git@bitbucket.imp.ac.at:7991/VBC/orgt.vbc.ac.at.git",`` ``"vbc.jenkins.buildnumber": "23",`` ``"vbc.jenkins.buildtag": "jenkins-SD-VBC-orgt.vbc.ac.at-develop-23",`` ``"vbc.jenkins.buildurl": "https://jenkins-impimba-1.imp.ac.at/job/SD/job/VBC/job/orgt.vbc.ac.at/job/develop/23/",`` ``"vbc.jenkins.node": "it-builder-2.vbc.ac.at",`` ``"vcs-ref": "4fdc108189b0b4eb06b7c61b270409218e25d81a",`` ``"vcs-type": "git",`` ``"vendor": "Red Hat, Inc.",`` ``"version": "1"``}`
```

## available parameters

**minimal call to buildDockerImage()**

```
`buildDockerImage([imageName: "myImage"])`
```

full list of parameters

| parameter               | type         | default                                            | description                                                  |
| :---------------------- | :----------- | :------------------------------------------------- | :----------------------------------------------------------- |
| imageName               | string       |                                                    | name for the container image built from ./Dockerfile with the docker build context in the root of the repository (i.e. "docker build ." in repo root) |
| buildExtraArgs          | string       | ""                                                 | extra (arbitrary) arguments to be added to the "docker build" command |
| pullRegistry            | string       | "[registry.redhat.io](http://registry.redhat.io/)" |                                                              |
| pullRegistryCredentials | string       | "redhat-registry-service-account"                  | Jenkins Credentials ID for the pull registry (that might require authentication) |
| pushBranches            | List         | ["develop"]                                        | which branches should also be pushed (the docker image is tagged with the name of the branch) |
| pushRegistry            | string       | docker.artifactory.imp.ac.at                       | i.e. a tag will be pushed to {pushRegistry}/{pushRegistryNamespace}/{imageName}:latest |
| pushRegistryNamespace   | string       | it                                                 |                                                              |
| pushRegistryCredentials | string       | jenkins_artifactory_creds                          |                                                              |
| pushBranches            | List         | ["develop"]                                        | which branches should also be pushed (the docker image is tagged with the name of the branch) |
| containerImages         | List of Maps | [ ]                                                | additional Dockerfiles that should be built to images, example:**extra container image configs**`containerImages = [``  ``[imageName: "my-extra-img", dockerFile: "relative/to/Dockerfile", dockerContext: "relative/to/context"],``  ``[imageName: "third-extra-img", dockerFile: "relative/third/Dockerfile", dockerContext: "relative/third/context"]``]`corresponding to `docker build --tag $imageName:the_tag --file $dockerFile $dockerContext``the_tag` will be determined by the git tag or the branch name. The per image config can also override push and pull registry configssee implementation: all imageConfig parameters: https://bitbucket.imp.ac.at/projects/IAB/repos/vbc-cicd/browse/vars/buildDockerImage.groovy#72 |
| test                    | Closure      | null                                               | Test implementation as closure (see section below).for an extended usage example see: https://bitbucket.imp.ac.at/projects/CLIP/repos/platinum/browse/Jenkinsfile |
| tower                   | Map          | [ ]                                                | tower jobs config, see section below.                        |
| agentLabels             | List         | ["dockerce", "rhel8"]                              | which build hosts the job should be scheduled on             |



## Testing

- optional, for all containers (1 test block for all images)
- Closure
- explain entryPoint: allBuilds Map
- convenience function testScript(script.sh, pattern)

Test implementation as closure (see section below). Convenience function: testScript('./test.sh', '**/TEST*.xml') will return a closure that calls ./test.sh within default container image, and collect tests from **/TEST*.xml

The test closure will only be executed once, it takes 2 parameters: (`imageName, dockerBuilds`)

imageName: name of the default image (see 1st parameter)

dockerBuilds: dict of dicts, image names as keys.

Example:

**allBuilds docker configs**

```groovy
`dockerBuilds = [`` ``myImage: [``  ``config: [ ... dict of various config params: imageName, dockerFile, dockerContext, registry infos],``  ``image: `` ``]``]`
```

for an extended usage example see: https://bitbucket.imp.ac.at/projects/CLIP/repos/platinum/browse/Jenkinsfile

## Ansible Tower

Depending on wether a tag or a specific branch was built, a Tower job will be triggered.

What builds are pushed



A defined Tower job will always be executed, except when one of the previous stages failed. Therefore also UNSTABLE build results will be pushed to the registry.

A possibility to avoid this behaviour is to fail the build explicitly after collecting the test results (in the testing Closure).



**Example: Tower jobs configs**

```groovy
`def` `towerJobs = [`` ``tags:  [jobName:``"My App Prod"``, jobTags: ``"reload"``, extraVars: ``"image_tag: latest"``],`` ``develop: [jobName:``"My App Testing"``, jobTags: ``"reload"``, extraVars: ``"image_tag: develop"``],`` ``new_feature:  [jobName:``"My App Dev"``, jobTags: ``"reload"``, extraVars: ``"image_tag: feat1"``]``]`
```

The job under the "tags" key will be used to trigger jobs after a git tag was build. 

All other keys are used to reference branch names (and trigger jobs accordingly).

In the config "jobName" is required. Optional parameters include jobTags (Ansible tags passed to the job execution), "extraVars" (raw yaml) passed in as ansible -e extravars. It is also possible to set different limits or inventories (not recommended).

For all parameters see implementation here: [source of runTowerJob.groovy](https://bitbucket.imp.ac.at/projects/IAB/repos/vbc-cicd/browse/vars/runTowerJob.groovy?at=refs%2Fheads%2Fmaster)

## Examples

### push registry settings

i.e. a tag will be pushed to `{pushRegistry}/{pushRegistryNamespace}/{imageName}:latest`

### more complex Jenkinsfile

The example is taken from Platinum application: https://bitbucket.imp.ac.at/projects/CLIP/repos/platinum/browse/Jenkinsfile

1. define testing in "platinumTest" closure
2. define deployment with "towerJobs"
3. build additional images with "extraImages"
4. put it all together in "buildDockerImage" call

**Jenkinsfile** Collapse source

```groovy
`#!/usr/bin/env groovy` `// test implementation for Platinum application``def` `platinumTest = { defaultImageName, allBuilds ->`` ``def` `db_user = ``'platinum'`` ``def` `db_pass = ``'test'`` ``def` `db_name = ``'platinum_test'` ` ``// create database, setup DB user access`` ``def` `db_cont = allBuilds.db.image.run(``"--env 'POSTGRES_USER=${db_user}' --env 'POSTGRES_PASSWORD=${db_pass}' --env 'POSTGRES_DB=${db_name}'"``)` ` ``// give container time to do full db init`` ``sleep time: ``10``, unit: ``'SECONDS'`` ``echo ``"started db container, id: ${db_cont.id}"` ` ``def` `db_uri = ``"postgresql+psycopg2://${db_user}:${db_pass}@${db_cont.id}/${db_name}"`` ``// link app container to db container, expose platinum test config path`` ``echo ``"db_uri: ${db_uri}"`` ``// prepare extra oidc secrets`` ``def` `secretsFile = ``"${WORKSPACE}/oidc_client_secrets.yaml"`` ``withCredentials([usernamePassword(credentialsId: ``'platinum-adfs-client-config'``, usernameVariable: ``'OIDC_CLIENT_ID'``, passwordVariable: ``'OIDCS_CLIENT_SECRET'``)]) {``   ``def` `oidc_secrets = [``     ``adfs_client_id: ``"${env.OIDC_CLIENT_ID}"``,``     ``adfs_client_secret: ``"${env.OIDCS_CLIENT_SECRET}"``   ``]``   ``echo ``"oidc secrets in ${secretsFile}"``   ``writeYaml file: secretsFile, data: oidc_secrets`` ``}` ` ``// run actual tests`` ``try` `{``   ``allBuilds.webservice.image.inside(``"--link ${db_cont.id} -e PLATINUM_DATABASE_URI=${db_uri} -e OIDC_CLIENT_SECRETS=${secretsFile}"``) {` `     ``// run the behave tests``     ``echo ``"running behave tests"``     ``sh returnStatus: true, script: ``""``"cd /srv/webservice/``       ``behave --junit --junit-directory ${env.WORKSPACE}``     ``""``"``   ``}`` ``}`` ``catch` `(exc) {``  ``echo ``"something bad happened"``  ``echo ``"${exc}"`` ``}`` ``finally` `{``  ``// ensure db container goes away``  ``echo ``"stopping db container"``  ``db_cont.stop()` `  ``// collect behave test results``  ``junit ``"TESTS-*.xml"`` ``}``}` `// tower jobs to trigger after build/push``def` `towerJobs = [`` ``tags:  [jobName:``"App Platinum Prod"``, jobTags: ``"reload"``, extraVars: ``"image_tag: latest"``],`` ``develop: [jobName:``"App Platinum Dev"``, jobTags: ``"reload"``, extraVars: ``"image_tag: develop"``],`` ``clip:  [jobName:``"App Platinum Dev"``, jobTags: ``"reload"``, extraVars: ``"image_tag: clip"``]``]` `// additional images to build, (same namespace, tag, registry as above, tests default to global params) list of dicts:``def` `extraImages =`` ``[``  ``[imageName: ``"db"``, dockerFile: ``"./Dockerfile.db"``, dockerContext: ``"."``]`` ``]` `buildDockerImage([imageName: ``"webservice"``,``         ``test: platinumTest,``         ``pushRegistry: ``"clip-docker.artifactory.imp.ac.at"``,``         ``pushRegistryNamespace: ``"platinum"``,``         ``containerImages: extraImages,``         ``pushBranches: [``"develop"``, ``"clip"``],``         ``tower: towerJobs])`
```
