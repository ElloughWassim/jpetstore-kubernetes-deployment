// CI/CD pipeline for JPetStore (DevOps layer, not part of the upstream MyBatis project).
//
// Uses https://github.com/ElloughWassim/jenkins-pipeline-library registered in Jenkins as
// "jenkins-pipeline-library". See README.md > "CI/CD pipeline" for plugins, tools and credentials.
@Library('jenkins-pipeline-library@v1.0.0') _

def image

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 60, unit: 'MINUTES')
    }

    parameters {
        choice(name: 'REGISTRY', choices: ['dockerhub', 'ecr'], description: 'Container registry for the image')
        booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Deploy to Kubernetes after a successful build')
    }

    environment {
        // ---- Replace the placeholders (or define them as global Jenkins environment variables) ----
        DOCKERHUB_NAMESPACE = '<DOCKERHUB_NAMESPACE>'
        AWS_ACCOUNT_ID      = '<AWS_ACCOUNT_ID>'
        AWS_REGION          = 'eu-west-1'
        IMAGE_NAME          = 'jpetstore'
    }

    stages {
        stage('Unit tests') {
            // The upstream license-header plugin is skipped in CI, as in the upstream workflows.
            steps { mavenUnitTest(properties: ['license.skip': 'true']) }
        }

        stage('Integration tests') {
            // Upstream screen-transition tests (Selenide + HtmlUnit) against Tomcat 9 started by Cargo.
            steps {
                mavenIntegrationTest(profiles: 'tomcat9', skipUnitTestsProperty: null,
                                     properties: ['license.skip': 'true'])
            }
        }

        stage('SonarQube analysis') {
            steps {
                sonarQubeAnalysis(installationName: 'sonarqube', projectKey: 'jpetstore-kubernetes-deployment',
                                  properties: ['license.skip': 'true'])
            }
        }

        stage('Quality gate') {
            steps { sonarQubeQualityGate(timeoutMinutes: 10) }
        }

        stage('OWASP Dependency-Check') {
            steps {
                owaspDependencyCheck(odcInstallation: 'dependency-check', nvdApiKeyCredentialsId: 'nvd-api-key',
                                     failOnCvss: 7)
            }
        }

        stage('Trivy filesystem scan') {
            steps { trivyScan(type: 'fs', target: '.') }
        }

        stage('Docker build') {
            steps {
                script {
                    Map imageConfig = params.REGISTRY == 'ecr'
                        ? [registry: 'ecr', awsAccountId: env.AWS_ACCOUNT_ID, awsRegion: env.AWS_REGION,
                           name: env.IMAGE_NAME, tags: imageTags(extraTags: [])]
                        : [registry: 'dockerhub', namespace: env.DOCKERHUB_NAMESPACE,
                           name: env.IMAGE_NAME, tags: imageTags(extraTags: ['latest'])]
                    image = dockerBuild(image: imageConfig, dockerfile: 'deploy/docker/Dockerfile')
                }
            }
        }

        stage('Trivy image scan') {
            steps { trivyScan(type: 'image', target: image.primaryReference) }
        }

        stage('Push image') {
            steps {
                script {
                    if (params.REGISTRY == 'ecr') {
                        dockerPush(image: image, awsCredentialsId: 'aws-credentials')
                    } else {
                        dockerPush(image: image, credentialsId: 'dockerhub-credentials')
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when { expression { params.DEPLOY } }
            steps {
                kubernetesDeploy(manifests: 'deploy/kubernetes', namespace: 'jpetstore', deployment: 'jpetstore',
                                 image: image.primaryReference, kubeconfigCredentialsId: 'kubeconfig',
                                 rolloutTimeout: '300s')
            }
        }
    }

    post {
        always {
            script {
                if (image) {
                    dockerCleanup(image: image)
                }
            }
        }
    }
}
