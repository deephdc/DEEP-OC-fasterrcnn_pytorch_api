#!/usr/bin/groovy

@Library(['github.com/indigo-dc/jenkins-pipeline-library@1.4.0']) _

pipeline {
    agent {
        label 'docker-build'
    }

    environment {
        dockerhub_repo = "deephdc/deep-oc-fasterrcnn_pytorch_api"
        // in the case of pytorch, it is the same base pytorch image for CPU and GPU
        base_tag = "1.13.1-cuda11.6-cudnn8-runtime"
    }

    stages {
        stage('Validate metadata') {
            steps {
                checkout scm
                sh 'deep-app-schema-validator metadata.json'
            }
        }
        stage('Docker image building') {
            when {
                anyOf {
                    branch 'master'
                    branch 'test'
                    buildingTag()
                }
            }
            steps{

                dir('check_oc_artifact'){
                    // clone checking scripts
                    git url: 'https://github.com/deephdc/deep-check_oc_artifact'
                }

                dir('deep-oc-user_app'){
                    checkout scm
                    script {
                        // build different tags
                        id = "${env.dockerhub_repo}"

                        if (env.BRANCH_NAME == 'master') {
                           // CPU (aka latest, i.e. default)
                           // GPU : pytorch allows to run same image on CPU or GPU
                           id_cpu_gpu = DockerBuild(id,
                                            tag: ['latest', 'cpu', 'gpu'],
                                            build_args: ["tag=${env.base_tag}",
                                                         "branch=master"])

                           // Check that the image starts and get_metadata responses correctly
                           sh "bash ../check_oc_artifact/check_artifact.sh ${env.dockerhub_repo}"
                        }

                        if (env.BRANCH_NAME == 'test') {
                           // CPU + GPU
                           id_cpu_gpu = DockerBuild(id,
                                            tag: ['test', 'cpu-test', 'gpu-test'],
                                            build_args: ["tag=${env.base_cpu_tag}",
                                                         "branch=test"])

                           // Check that the image starts and get_metadata responses correctly
                           sh "bash ../check_oc_artifact/check_artifact.sh ${env.dockerhub_repo}:test"

                        }
                    }
                }
            }
            post {
                failure {
                    DockerClean()
                }
            }
        }

        stage('Docker Hub delivery') {
            when {
                anyOf {
                   branch 'master'
                   branch 'test'
                   buildingTag()
               }
            }
            steps{
                script {
                    DockerPush(id_cpu_gpu)
                }
            }
            post {
                failure {
                    DockerClean()
                }
                always {
                    cleanWs()
                }
            }
        }

        stage("Render metadata on the marketplace") {
            when {
                allOf {
                    branch 'master'
                    changeset 'metadata.json'
                }
            }
            steps {
                script {
                    def job_result = JenkinsBuildJob("Pipeline-as-code/deephdc.github.io/pelican")
                    job_result_url = job_result.absoluteUrl
                }
            }
        }
    }
}
