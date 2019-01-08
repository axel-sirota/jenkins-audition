#!/usr/bin/env groovy

def testPypi = 'https://test.pypi.org/legacy/'
def String imageName = "axelsirota/pluralsight-audition/jenkins-sample"
def String dockerArguments = "-it -v ${env.WORKSPACE}/reports:/reports -p 5000:5000"
def appImage

pipeline {
    agent any
    stages {
        stage('Setup Environment') {
            steps {
                sh '''#!/bin/bash
                    python3 -m pip install virtualenv
                    mkdir -p reports 
                ''' 
            }
        }
        stage ('Build Image') {
            steps {
                script {
                    appImage = docker.build("${imageName}:0.${env.BUILD_ID}", "-f ${env.WORKSPACE}/docker/Dockerfile .")
                }
            }
        }
        stage('Compile') {
            steps {
                script {
                    appImage.withRun(dockerArguments){
                        "python -m  compileall -f app"
                    }
                }
            }
        }
        stage('Build and install'){
            steps {
                script {
                    appImage.withRun(dockerArguments){
                        "python setup.py bdist_wheel"
                        "python setup.py install"
                    }
                }
            }
        }
        stage('Run Tests') {
            steps {
                script {
                    appImage.withRun(dockerArguments){
                        "python setup.py nosetests"
                    }
                }
                step([$class: 'JUnitResultArchiver', testResults: 'reports/tests.xml'])
                step([$class: 'CoberturaPublisher', autoUpdateHealth: false, autoUpdateStability: false, coberturaReportFile: 'reports/coverage.xml', failUnhealthy: true, failUnstable: true, maxNumberOfBuilds: 0, onlyStable: true, sourceEncoding: 'ASCII', zoomCoverageChart: true])
            }
        }
        stage('Code Checking') {
            steps {
                script {
                    appImage.withRun(dockerArguments){
                        "python -m pylint app --exit-zero >> reports/pylint.log"
                        "python -m flake8 app --exit-zero"
                    }
                }
                step([
                    $class                     : 'WarningsPublisher',
                    parserConfigurations       : [[parserName: 'PYLint', pattern   : 'output/pylint.log']],
                    unstableTotalAll           : '20',
                    usePreviousBuildAsReference: true
                ])
                step([
                    $class                     : 'WarningsPublisher',
                    parserConfigurations       : [[parserName: 'Flake8', pattern   : 'output/flake8.log']],
                    unstableTotalAll           : '20',
                    usePreviousBuildAsReference: true
                ])
            }
        }
        stage('Archive reports') {
            steps {
                archive 'reports/*'
            }
        }
        stage('Decide to deploy to Docker Hub') {
            agent none
            steps {
                script {
                env.TAG_ON_DOCKER_HUB = input message: 'User input required',
                    parameters: [choice(name: 'Deploy to Docker Hub', choices: 'no\nyes', description: 'Choose "yes" if you want to deploy this build')]
                }
            }
        }
        stage('Deploy to Docker Hub') {
            agent any
            when {
                environment name: 'TAG_ON_DOCKER_HUB', value: 'yes'
            }
            steps {
                script {
                    docker.withRegistry('', 'dockerhub-key') {
                        appImage.push()
                    }
                }
            }
        }
    }
}
