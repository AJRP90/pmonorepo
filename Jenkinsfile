#!/usr/bin/env groovy
// Load the shared libraries
//@Library('jenkins-shared-libraries')_

import static groovy.io.FileType.FILES

// Load child Jenkinsfiles based on diff
def loadDiff() {
  dirs = []
  loads = [:]
  homeChange = false
  files = findFiles(glob: '**/Jenkinsfile')
  matches = sh(returnStdout:true, script: "git log -p -1 | grep "diff --git" | grep -v a/Jenkinsfile | cut -d '/' -f 2 | uniq")
  match_list = matches.tokenize('\n')
  match_list.each { match ->
    if (match.tokenize('/').size() == 1) {
      homeChange = true
    } else {
      dirs << match.tokenize('/').take(1).join('/') // Each service should be in its own directory within monorepo
    }
  }
  if (homeChange) {
    loadAll()
  } else {
    uniqDirs = dirs.unique()
    uniqDirs.each() { dir ->
      files.each() { file ->
        if (file.path =~ dir) {
          loads["Service ${dir}"] = { load file.path }
        }
      }
    }
    loads.failFast = false // Build other services even if one of them failed
    parallel loads         // Build all services in parallel
  }
}

// Load all Jenkinsfiles within the repo for master branch builds, etc.
def loadAll() {
  loads = [:]
  files = findFiles(glob: '**/Jenkinsfile')
  files.each() { file ->
    if (file.path.tokenize('/').size() != 1) {
      name = file.path.replaceAll('/Jenkinsfile', '') // Assuming service directory name == service name
      loads["Service ${name}"] = { load file.path }
    }
  }
  loads.failFast = false // Build other services even if one of them failed
  parallel loads         // Build all services in parallel
}

// Stop running build for PullRequest if something new was pushed. Just like Travis-CI does.
def abortPreviousBuilds() {
  /*
  Author: Isaac S Cohen
  Modification: Abort only previous builds
  */

  jobname = env.JOB_NAME
  buildnum = env.BUILD_NUMBER.toInteger()

  job = Jenkins.instance.getItemByFullName(jobname)
    for (build in job.builds) {
      if (buildnum > build.getNumber().toInteger()) { build.doStop(); }
  }
}

// Check if build for Tag is in progress and abort the build to avoid race conditions.
def stopOnDeploy() {
  /*
  Check if Another Tag build is in progress
  */
  identifier = "${env.TAG_NAME}".replaceAll("[0-9]+|-", "") + "-[0-9]+"
  runningJobs = Jenkins.instance.getView('All').getBuilds().findAll() { it.getResult().equals(null) }

  runningJobs.each { job ->
    if (job.getParent().getDisplayName() =~ identifier && job.getParent().getDisplayName() != "${env.TAG_NAME}" && job.getId().toInteger() < "${env.BUILD_NUMBER}".toInteger()) {
      println "Someone deploying Prod. Aborting"
      currentBuild.result = 'ABORTED'
      error('Another Prod deploy is in progess. Aborting')
    }
  }
}

node {
  cleanWs disableDeferredWipeout:true                // clean workspace before checkout. Workspace-cleanup plugin is required
  checkout scm
  try {
    if (env.CHANGE_ID != null) {
      abortPreviousBuilds()
      loadDiff()
    } else {
      loadAll()
    }
    if (env.BRANCH_NAME == 'master') {
      slackSend (color: '#28aa78', message: "SUCCESSFUL: Job '${env.JOB_NAME} for '${env.BRANCH_NAME}' [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
    } else if (env.BRANCH_NAME =~ 'PR-.*') {
      slackSend (color: '#28aa78', message: "SUCCESSFUL: Congrats, ${env.CHANGE_AUTHOR}! Job '${env.JOB_NAME} for 'PR-${env.CHANGE_ID}' [${env.BUILD_NUMBER}]' (${env.BUILD_URL}) passed")
    }
  } catch(err) {
    if (env.BRANCH_NAME == 'master') {
      slackSend (color: '#a00000', message: "FAILED: Job '${env.JOB_NAME} for '${env.BRANCH_NAME}' [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
    } else if (env.BRANCH_NAME =~ 'PR-.*') {
      slackSend (color: '#a00000', message: "FAILED: Hey! ${env.CHANGE_AUTHOR}! Job '${env.JOB_NAME} for 'PR-${env.CHANGE_ID}' [${env.BUILD_NUMBER}]' (${env.BUILD_URL}) failed!")
    }
    throw err
  } finally {
    cleanWs disableDeferredWipeout:true                // clean workspace before checkout. Workspace-cleanup plugin is required
  }
}