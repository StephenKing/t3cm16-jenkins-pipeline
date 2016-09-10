class: center, middle, inverse

# Continuous Delivery

## using Jenkins and the Pipeline Plugins


Steffen Gebert (@StGebert, first.last@typo3.org)

Available at https://github.com/StephenKing/t3cm16-jenkins-pipeline

---
# /me

* Researcher & PhD Student at University of Würzburg (Communication Networks)
* Member of the _TYPO3 Server Admin Team_

---
class: center, middle, inverse
# (Why) Continuous Delivery


???

- Wer weiß halbwegs Bescheid, was mit CD gemeint ist?

- Rest Hand heben - Wer arbeitet in einem agilen Team?

- Wer verwendet Jenkins?

- Wer verwendet Pipelines in irgendeiner Form



---
# Continuous Delivery


- Reduce cycle times
  - Shorter time-to-market
  - Faster feedback

- Plays very well with agile development
  - Agile Manifesto: _"Our highest priority is to satisfy the customer through early and continuous delivery of valuable software."_


![:scale 70%](img/cycles.png)


---
# Pipelines

![:scale 100%](img/pipeline.png)

- Minimize execution time- Always aware of latest _stable_ release
- Every checkin triggers pipeline execution

---
# Feedback![:scale 90%](img/pipeline-feedback.png)
- Feedback to the team in every stage
  - Bring the pain forwad
  - Fail fast, fail often
  - Remember the Gamification talk!?

---
class: center, middle, inverse
# CI/CD Tools

---
# CI/CD Tools

- On-premise
  - *Jenkins*
  - Thoughtworks Go
  - Gitlab CI
  
- SaaS
  - TravisCI
  - CircleCI
  - AppVeyor
  - Codeship
  - Visual Studio Team Services

???

Welche Tools habt ihr verwwendet?

---
# Why (I like) Jenkins

- Established open source project
 
- On premise installation

- Thousands of plugins


---
# History of CI/CD with Jenkins 

- Downstream Jobs
  - Job A triggers job B triggers job C triggers job D trig...
  - If one job fails, fail all
  
- _Build Pipeline_ View

  ![:scale 100%](img/build-pipeline.png)



---
class: center, middle, inverse
# But
## that's awkward!

---
# Configuration as Code

- Define jobs/pipelines as code
  - Avoid point & click  
  - **In version control**
  - Lives with the application
  - **Scales better**

- Example: `.travis.yml` (from TravisCI)

```yaml
language: php
services:
  - redis-server
before_script:
  - composer install
script:
  - ./bin/phpunit -c typo3/sysext/core/Build/UnitTests.xml
notifications:
  slack:
    ...
```

???

Wenn man manuell die Jobs für neue Projekte einrichten muss, macht man's ja eher nicht.

---
# Previous Approaches in Jenkins

- Jenkins Job Builder
  - Python / YAML based, from the OpenStack project

- Job DSL plugin
  - Groovy DSL! (e.g. query Github API and create jobs for all branches)
  - Still creates single jobs

```groovy
job('my-project-main') {
    scm {
        git('https://github.com/...')
    }
    triggers {
        scm('H/15 * * * *')
    }
    publishers {
        downstream('my-project-unit')
    }
}
```  

  
---
class: center, middle, inverse

### There is relief
# Jenkins Pipeline Plugins

---
# Jenkins Pipeline Plugins

- Whole suite of plugins (10+)
  - Open-sourced earlier this year
  - Shipped with Jenkins 2.0
  - Formerly commercially available by CloudBees, called _Workflow_
  - Define your pipeline as code (again Groovy DSL)

```groovy
stage(name: "Hello") {
    echo "*Hello*"
}
stage(name: "World") {
    echo "*World*"
}
```

![:scale 70%](img/hello-world.png)

---
# Pipeline Execution

- Jenkins executors (configured, limited number)
  - Heavyweight (run on any _node_ (master, slave), blocks executor)
  - Flyweight (running on master, "for free")

```groovy
node {
    stage(name: "Build") {
        sh "echo Could run 'composer intall' now"
    }
    stage(name: "Unit") {
        sh "echo Could run 'phpunit' now"
    }     // ...
}
```
![:scale 70%](img/example-output.png)

---
# Pipeline Steps

- Online docs: https://jenkins.io/doc/pipeline/steps/
- Allocate a node
  - `node`: Allocate a node to execute job
  - `node('php')`: Allocates a node (with PHP installed)
- Model pipeline visualization `stage`
- Get your code: `checkout "https://github.com/.."`
- Execute commands: `sh` (*nix), `bat` (Windows)  
- File Handling: `readFile`, `writeFile`, `fileExists`
- Execute jobs in parallel: `parallel`
- General build step (including parameters): Call any (compatible) plugin
```
step([$class: 'ArtifactArchiver', artifacts: '*.jar'])
```
- Call JobDSL plugin: `jobdsl`


---
# Docker

- Run build jobs within Docker containers
  - No need to install software on Jenkins master/slave
  - Allows to use multiple versions of the same tool

```groovy
stage("run in docker") {
    node {
       withDockerContainer("php:7-fpm") {
           sh "php -v"
       }
    }
}
```

- Containers can be existing ones ore built on demand

---
# Snippet Editor & Docs

- Because first steps are hard..

  ![:scale 100%](img/snippet-editor.png)

- Auto-generated DSL documentation (_Pipeline Syntax_ → _Step Reference_)

---
# Jenkinsfile

- File called `Jenkinsfile` contains pipeline code

- Pipeline code co-located with application code
  - It is versioned
  - Everybody can read (and modify) it

- Throw away your Jenkins installation at any time

- But: Lacks reusability

---
# Multibranch & Organization Folders

- Scans a complete GitHub/Bitbucket organization for `Jenkinsfile`
  - Triggered by Webhook and/or runs periodically
  - Automatically adds pipeline jobs per repo/branch/PR

![:scale 100%](img/orgfolder.png)

---
# Jenkins Global Library

- Provdies shared functionality available for all jobs
- Stored on Jenkins master, available to all slaves
- Available via Jenkins-integrated Git server

- Add your own (Groovy) code

```groovy
package org.foo
class Utilities {
  def steps
  Utilities(steps) {this.steps = steps}
  def mvn(args) {
    steps.sh "${steps.tool 'Maven'}/bin/mvn -o ${args}"
  }
}
```


```groovy
@Library('utils') import org.foo.Utilities
def utils = new Utilities(steps)
node {
  utils.mvn 'clean package'
}
```
---
class: center, middle, inverse
# Real-World Example

---
# Real-World Example


- Chef CI/CD at TYPO3.org: Code that runs the *.typo3.org infrastructure
  - https://chef-ci.typo3.org

- Objective: Chef cookbooks
  - Server provisioning (installs packages, configures services) 
  - Code: https://github.com/TYPO3-cookbooks

![:scale 100%](img/chefci-stages.png)

---
# Real-World Example (2)

- Scans our GitHub organization _TYPO3-cookbooks_
  - Automatically adds/removes pipelines for branches and pull requests
  - Triggered via Webhooks
  - Searches for `Jenkinsfile`
```groovy
def pipe = new org.typo3.chefci.v1.Pipeline()
pipe.execute()
```

- Uses Jenkins Global Library
  - Code: [TYPO3-infrastructure/jenkins-pipeline-global-library-chefci](https://github.com/TYPO3-infrastructure/jenkins-pipeline-global-library-chefci)
  ![:scale 80%](img/global-library.png)

- Only _master_ branch uploads artifact (to Chef Server)
  - (historically using _Git Flow_) 

---
class: center, middle, inverse
# Blue Ocean

### The Future of Jenkins UI

---
# Blue Ocean


- Tailored to the pipeline plugins

![:scale 100%](img/blue-overview.png)


- https://jenkins.io/projects/blueocean/
- https://jenkins.io/blog/2016/07/19/blue-ocean-update/

---
# Blue Ocean


- Tailored to the pipeline plugins

![:scale 100%](img/blue-detail.png)

---
# Thanks to Our T3CM16 Sponsors!

### Platin

- jweiland.net
- punkt.de
- HISCOX
- Mittwald

### Gold

- e-central
- in2code
- Arrabiata


### Silver

- Spooner.Web, aimeos, pixelink, Sign&Sinn, SGALINSKI, datamints, mindscreen

---
# Summary

- Continuous Delivery streamlines and speeds up development

- Jenkins pipeline suite modernizes its CI/CD capabilities
  - CloudBees, Inc. is very actively pushing development
  - Some parts still pretty tricky
  - Many Jenkins plugins already compatible

- Pipeline defined as code
  - Versioned
  - Doesn't mess up Jenkins jobs
  - Code sharing
  
- Automated pipeline creation based on GitHub/Bitbucket APIs
 
- Blue Ocean refreshes Jenkins' UI