pipeline {
    agent any
    parameters {
        choice(name: 'QualityGate', choices: ['Pass', 'Fail'], description: 'Choose the QualityGate status (Pass or Fail). The pipeline will fail if "Fail" is selected.')
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_REGION = 'ap-south-1'
        ECR_REPO_API = '836759839628.dkr.ecr.ap-south-1.amazonaws.com/dev/web'
        IMAGE_TAG = "${COMMIT_ID}-${BUILD_NUMBER}"
    }
    stages {
        stage('Checkout Code') {
            steps {
                // Checkout the source code from the repository
                checkout scm
            }
        }

        stage('Static Code Analysis') {
            parallel {
                stage('SonarQube Analysis') {
                    steps {
                        script {
                            withSonarQubeEnv('sonar-scanner') {
                                sh '''
                                echo "Running SonarQube analysis for WEB"
                                $SCANNER_HOME/bin/sonar-scanner \
                                    -Dsonar.projectName=web \
                                    -Dsonar.projectKey=web \
                                    -Dsonar.qualitygate.name=SonarCustom
                                '''
                            }
                        }
                    }
                }

                stage('OWASP Dependency-Check Scan') {
                    steps {
                        dependencyCheck additionalArguments: '--scan ./ --format ALL', 
                                        odcInstallation: 'dp', 
                                        stopBuild: true
                        dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                echo "Quality Gate Check OK "
            }
        }

    //    stage('Quality Gate Check') {
    //        steps {
    //            script {
    //                // Wait for SonarQube Quality Gate result
    //                def qg = waitForQualityGate()  // This waits for the analysis to complete

    //                // Check if the quality gate passed or failed and take action based on user choice
    //                if (params.QualityGate == 'Fail' && qg.status != 'OK') {
    //                    // Fail the pipeline if the quality gate fails and 'Fail' is chosen
    //                    error "Quality Gate '${qg.name}' Failed. Build is failing as per user choice."
    //                } else if (params.QualityGate == 'Pass' && qg.status == 'OK') {
    //                    echo "Quality Gate passed. Continuing pipeline."
    //                } else if (params.QualityGate == 'Pass' && qg.status != 'OK') {
    //                    // Optionally, continue even if the Quality Gate failed if 'Pass' was selected
    //                    echo "Quality Gate failed, but continuing because 'Pass' was selected."
    //                }
    //            }
    //        }
    //    }

        stage('Run WEB Tests') {
            steps {
                script {
                    dir('web') {
                        sh 'export NODE_OPTIONS=--openssl-legacy-provider && npm install'  // Install dependencies
                        sh 'export NODE_OPTIONS=--openssl-legacy-provider && npm run test:unit'  // Run unit tests for the web app
                        echo "run web test"
                    }
                }
            }
        }

        stage('Build WEB Docker Image') {
            steps {
                script {
                    dir('web') {
                        // Build Docker image for the API
                        sh 'docker build -t $ECR_REPO_API:$IMAGE_TAG .'
                    }
                }
            }
        }

        stage('Anchore Grype Vulnerability Scan') {
            steps {
                script {
                    // Run Grype scan on the API Docker image (without condition)
                    sh 'grype $ECR_REPO_API:$IMAGE_TAG -o json > grype-scan-api.json'

                    // Display the Grype scan results
                    sh 'cat grype-scan-api.json'

                    // Optionally, you can archive the report to keep it for later inspection
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'grype-scan-api.json'
                }
            }
        }

        stage('Push API Docker Image to ECR') {
            steps {
                script {
                    // Log in to AWS ECR and push the API Docker image
                    sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin 836759839628.dkr.ecr.$AWS_REGION.amazonaws.com'
                    sh 'docker push $ECR_REPO_API:$IMAGE_TAG'
                }
            }
        }
    }
}
