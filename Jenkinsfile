// Pipeline final : qualité (en parallèle) -> build -> scan Trivy -> push (main) -> staging -> validation -> production
pipeline {
    agent any

    environment {
        IMAGE = 'christlk/legeni-webapp'
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()          // jamais deux déploiements en même temps
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    stages {
        stage('Préparation') {
            steps {
                script {
                    env.BASE_VERSION = readFile('VERSION').trim()
                    env.TAG = "${env.BASE_VERSION}-${env.BUILD_NUMBER}"
                    currentBuild.displayName = "#${env.BUILD_NUMBER} (${env.TAG})"
                }
                echo "Version ${env.TAG} sur la branche ${env.BRANCH_NAME}"
            }
        }

        stage('Qualité') {
            failFast true
            parallel {
                stage('Tests unitaires') {
                    agent {
                        docker {
                            image 'python:3.12-slim'
                            args '-e HOME=/tmp -v pip_cache:/tmp/.cache/pip'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            python -m venv /tmp/venv
                            /tmp/venv/bin/pip install -q --default-timeout=120 --retries 5 -r requirements.txt -r requirements-dev.txt
                            mkdir -p reports
                            /tmp/venv/bin/pytest --junitxml=reports/junit.xml \
                                --cov=app --cov-report=xml:reports/coverage.xml --cov-fail-under=80
                        '''
                    }
                    post {
                        always {
                            junit 'reports/junit.xml'
                            recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
                        }
                    }
                }
                stage('Lint Python') {
                    agent {
                        docker {
                            image 'python:3.12-slim'
                            args '-e HOME=/tmp'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            python -m venv /tmp/venv
                            /tmp/venv/bin/pip install -q --default-timeout=120 --retries 5 flake8==7.1.1
                            mkdir -p reports
                            /tmp/venv/bin/flake8 --output-file=reports/flake8.txt --tee .
                        '''
                    }
                    post {
                        always {
                            recordIssues(tools: [flake8(pattern: 'reports/flake8.txt')])
                        }
                    }
                }
                stage('Lint Dockerfile') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint:v2.12.0 < Dockerfile'
                    }
                }
                stage('Syntaxe Ansible') {
                    steps {
                        withCredentials([file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_FILE')]) {
                            dir('ansible') {
                                sh '''
                                    export ANSIBLE_CONFIG="$PWD/ansible.cfg"
                                    ansible-playbook deploy.yml --syntax-check -e app_version=0.0.0 --vault-password-file "$VAULT_FILE"
                                    ansible-playbook provision.yml --syntax-check
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$TAG -t $IMAGE:$TAG .'
            }
        }

        stage('Scan de sécurité') {
            steps {
                // --exit-code 1 : échec si une vulnérabilité CRITICAL corrigeable est trouvée
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy_cache:/root/.cache \
                        aquasec/trivy:0.57.1 image --severity CRITICAL --ignore-unfixed --exit-code 1 $IMAGE:$TAG
                '''
            }
        }

        stage('Push Docker Hub') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$TAG
                        docker logout
                    '''
                }
            }
        }

        stage('Déploiement staging') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                                 file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_FILE')]) {
                    dir('ansible') {
                        sh '''
                            export ANSIBLE_CONFIG="$PWD/ansible.cfg"
                            ansible-playbook deploy.yml -e target_env=staging -e app_version=$TAG -e app_image=$IMAGE \
                                --vault-password-file "$VAULT_FILE"
                        '''
                    }
                }
            }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: "Déployer la version ${env.TAG} en production ?", ok: 'Déployer'
                }
            }
        }

        stage('Déploiement production') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                                 file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_FILE')]) {
                    dir('ansible') {
                        sh '''
                            export ANSIBLE_CONFIG="$PWD/ansible.cfg"
                            ansible-playbook deploy.yml -e target_env=prod -e app_version=$TAG -e app_image=$IMAGE \
                                --vault-password-file "$VAULT_FILE"
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "OK ${env.JOB_NAME} #${env.BUILD_NUMBER} : version ${env.TAG}"
        }
        failure {
            echo "ECHEC ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            withCredentials([string(credentialsId: 'webhook-discord', variable: 'HOOK')]) {
                sh 'curl -s -H "Content-Type: application/json" -d "{\\"content\\": \\"ECHEC $JOB_NAME #$BUILD_NUMBER - $BUILD_URL\\"}" "$HOOK"'
            }
        }
        cleanup {
            cleanWs()
        }
    }
}