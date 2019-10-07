#!groovy​

pipeline {
    agent any

    environment {
        PRODUCTION_BRANCH = 'master'
        DEVELOP_BRANCH = 'develop'

        DOCKER_REPOSITORY = 'docker.dxc.com:80'
        DOCKER_REGISTRY = "${DOCKER_REPOSITORY}/phr"
        DOCKER_CREDENTIALS = 'phr-docker-registry'
        DOCKER_IMAGE = 'wso2is'
    }

    options {
        disableConcurrentBuilds()
    }

    stages {
        stage('Clone') {
            steps {
                checkout scm
            }
        }
			
		stage('Preparation'){
			git 'https://github.com/AJRP90/pmonorepo.git'
			//Get projects modified
			def mvnHome = git log -p -1 | grep diff | cut -d '/' -f 2 |uniq
			echo '$mvnHome'
		}

    }
}

def isDevelopBranch() {
    return env.BRANCH_NAME == env.DEVELOP_BRANCH
}

def isProductionBranch() {
    return env.BRANCH_NAME == env.PRODUCTION_BRANCH 
} 