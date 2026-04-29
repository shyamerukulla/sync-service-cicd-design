// =============================================================================
// Jenkinsfile — sync-service CI/CD Pipeline
// Environments: qa (develop) | staging (release/*) | prod (main)
// Strategy:     Blue/Green on GCP VMs via Ansible
// =============================================================================

pipeline {

    agent any

    // -------------------------------------------------------------------------
    // Global environment variables
    // -------------------------------------------------------------------------
    environment {
        APP_NAME        = 'sync-service'
        GAR_REGION      = 'asia-south1'                          // GCP region
        GAR_HOST        = "${GAR_REGION}-docker.pkg.dev"
        GCP_PROJECT     = credentials('gcp-project-id')          // Jenkins secret
        IMAGE_REPO      = "${GAR_HOST}/${GCP_PROJECT}/${APP_NAME}"
        IMAGE_TAG       = "${env.GIT_COMMIT[0..7]}"              // short SHA
        SONAR_HOST      = 'http://sonarqube.internal:9000'
    }

    // -------------------------------------------------------------------------
    // Build parameters — can be triggered manually too
    // -------------------------------------------------------------------------
    parameters {
        choice(
            name: 'FORCE_ENV',
            choices: ['', 'qa', 'staging', 'prod'],
            description: 'Override deploy target (leave blank for auto-detect)'
        )
    }

    // -------------------------------------------------------------------------
    // Options
    // -------------------------------------------------------------------------
    options {
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    // -------------------------------------------------------------------------
    // Stages
    // -------------------------------------------------------------------------
    stages {

        // ── Stage 0: Resolve environment from branch ──────────────────────────
        stage('Resolve Environment') {
            steps {
                script {
                    if (params.FORCE_ENV) {
                        env.DEPLOY_ENV = params.FORCE_ENV
                    } else if (env.BRANCH_NAME == 'main') {
                        env.DEPLOY_ENV = 'prod'
                    } else if (env.BRANCH_NAME.startsWith('release/')) {
                        env.DEPLOY_ENV = 'staging'
                    } else if (env.BRANCH_NAME == 'develop') {
                        env.DEPLOY_ENV = 'qa'
                    } else {
                        env.DEPLOY_ENV = 'none'   // PR / feature branch — no deploy
                    }

                    echo "Branch: ${env.BRANCH_NAME} → Deploy target: ${env.DEPLOY_ENV}"

                    // Credential ID per environment (separate GCP SAs)
                    env.GCP_SA_CRED = env.DEPLOY_ENV == 'prod'
                        ? 'gcp-sa-prod'
                        : env.DEPLOY_ENV == 'staging'
                            ? 'gcp-sa-staging'
                            : 'gcp-sa-qa'
                }
            }
        }

        // ── Stage 1: Checkout ──────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        // ── Stage 2: Build ─────────────────────────────────────────────────────
        stage('Build') {
            steps {
                sh '''
                    mvn clean package -DskipTests \
                        --batch-mode \
                        --no-transfer-progress
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        // ── Stage 3: Unit + Integration Tests ─────────────────────────────────
        stage('Test') {
            steps {
                sh '''
                    mvn verify \
                        --batch-mode \
                        --no-transfer-progress
                '''
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java'
                    )
                }
            }
        }

        // ── Stage 4: Code Quality ──────────────────────────────────────────────
        stage('SonarQube Scan') {
            // Skip on PRs if you bill per-analysis; enable to gate PRs too
            when {
                not { changeRequest() }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${APP_NAME} \
                            -Dsonar.host.url=${SONAR_HOST} \
                            --batch-mode
                    """
                }
                // Fail build if Quality Gate is not passed
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ── Stage 5: Docker Build + Push (skipped on PRs) ─────────────────────
        stage('Docker Build & Push') {
            when {
                expression { env.DEPLOY_ENV != 'none' }
            }
            steps {
                script {
                    def fullImage = "${IMAGE_REPO}:${IMAGE_TAG}"
                    def envImage  = "${IMAGE_REPO}:${env.DEPLOY_ENV}-latest"

                    withCredentials([file(credentialsId: env.GCP_SA_CRED, variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                        sh """
                            gcloud auth activate-service-account --key-file=\$GOOGLE_APPLICATION_CREDENTIALS
                            gcloud auth configure-docker ${GAR_HOST} --quiet

                            docker build \
                                --build-arg APP_ENV=${env.DEPLOY_ENV} \
                                --label git-commit=${IMAGE_TAG} \
                                --label branch=${env.BRANCH_NAME} \
                                -t ${fullImage} \
                                -t ${envImage} \
                                .

                            docker push ${fullImage}
                            docker push ${envImage}

                            echo ${IMAGE_TAG} > deploy-manifest.txt
                        """
                    }

                    // Store tag for rollback reference
                    archiveArtifacts artifacts: 'deploy-manifest.txt'
                }
            }
        }

        // ── Stage 6: Prod Approval Gate ───────────────────────────────────────
        stage('Approval — Prod') {
            when {
                expression { env.DEPLOY_ENV == 'prod' }
            }
            steps {
                script {
                    def approver = input(
                        message: "Deploy ${IMAGE_TAG} to PRODUCTION?",
                        submitterParameter: 'APPROVED_BY',
                        ok: 'Deploy',
                        parameters: [
                            string(
                                name: 'CHANGE_TICKET',
                                description: 'JIRA / change-management ticket number',
                                defaultValue: ''
                            )
                        ]
                    )
                    echo "Approved by: ${approver} | Ticket: ${approver.CHANGE_TICKET}"
                }
            }
        }

        // ── Stage 7: Deploy (Blue/Green via Ansible) ──────────────────────────
        stage('Deploy') {
            when {
                expression { env.DEPLOY_ENV != 'none' }
            }
            steps {
                script {
                    def deployImage = "${IMAGE_REPO}:${IMAGE_TAG}"

                    // Retrieve MongoDB URI from Google Secret Manager at deploy time
                    withCredentials([
                        file(credentialsId: env.GCP_SA_CRED, variable: 'GOOGLE_APPLICATION_CREDENTIALS'),
                        string(credentialsId: "mongo-uri-${env.DEPLOY_ENV}", variable: 'MONGO_URI')
                    ]) {
                        sh """
                            gcloud auth activate-service-account --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

                            ansible-playbook ansible/deploy.yml \
                                -i ansible/inventory/${env.DEPLOY_ENV}.ini \
                                -e "image=${deployImage}" \
                                -e "app_env=${env.DEPLOY_ENV}" \
                                -e "mongo_uri=\$MONGO_URI" \
                                -e "app_name=${APP_NAME}" \
                                --diff
                        """
                    }
                }
            }
        }

        // ── Stage 8: Smoke Test (post-deploy health check) ────────────────────
        stage('Smoke Test') {
            when {
                expression { env.DEPLOY_ENV != 'none' }
            }
            steps {
                script {
                    def healthUrl = getHealthUrl(env.DEPLOY_ENV)
                    echo "Running smoke test against: ${healthUrl}"

                    // Retry for up to 3 minutes (18 × 10s)
                    retry(18) {
                        sleep(time: 10, unit: 'SECONDS')
                        sh """
                            STATUS=\$(curl -s -o /dev/null -w '%{http_code}' ${healthUrl})
                            if [ "\$STATUS" != "200" ]; then
                                echo "Health check returned HTTP \$STATUS — retrying..."
                                exit 1
                            fi
                            echo "Health check passed: HTTP 200"
                        """
                    }
                }
            }
        }

    }   // end stages

    // -------------------------------------------------------------------------
    // Post actions
    // -------------------------------------------------------------------------
    post {

        success {
            script {
                if (env.DEPLOY_ENV != 'none') {
                    slackSend(
                        channel: "#deployments",
                        color: 'good',
                        message: "✅ *${APP_NAME}* `${IMAGE_TAG}` deployed to *${env.DEPLOY_ENV}* successfully.\n" +
                                 "Branch: `${env.BRANCH_NAME}` | Build: ${env.BUILD_URL}"
                    )
                }
            }
        }

        failure {
            script {
                // Attempt automated rollback if we reached the deploy stage
                if (env.DEPLOY_ENV && env.DEPLOY_ENV != 'none') {
                    echo "Build failed — attempting automated rollback..."
                    rollback(env.DEPLOY_ENV)
                }

                slackSend(
                    channel: "#deployments",
                    color: 'danger',
                    message: "❌ *${APP_NAME}* pipeline FAILED on *${env.DEPLOY_ENV}*.\n" +
                             "Branch: `${env.BRANCH_NAME}` | Build: ${env.BUILD_URL}\n" +
                             "Automated rollback attempted."
                )
            }
        }

        always {
            // Clean up local Docker images to avoid disk bloat on the Jenkins agent
            sh """
                docker image prune -f --filter label=git-commit=${IMAGE_TAG} || true
            """
            cleanWs()
        }

    }

}   // end pipeline

// =============================================================================
// Helper functions
// =============================================================================

/**
 * Returns the health-check URL for the given environment.
 * These map to internal GCP load balancer IPs or internal DNS.
 */
def getHealthUrl(String deployEnv) {
    def urls = [
        qa      : 'http://sync-service-qa.internal:8080/actuator/health',
        staging : 'http://sync-service-staging.internal:8080/actuator/health',
        prod    : 'http://sync-service-prod.internal:8080/actuator/health'
    ]
    return urls[deployEnv] ?: error("Unknown environment: ${deployEnv}")
}

/**
 * Automated rollback: re-deploys the last known-good image tag.
 * The previous tag is read from deploy-manifest.txt archived by the last
 * successful build of this job.
 */
def rollback(String deployEnv) {
    try {
        // Copy the artifact from the last successful build
        copyArtifacts(
            projectName: env.JOB_NAME,
            selector: lastSuccessful(),
            filter: 'deploy-manifest.txt',
            optional: true
        )

        def previousTag = readFile('deploy-manifest.txt').trim()

        if (!previousTag) {
            echo "No previous image tag found — skipping rollback."
            return
        }

        def previousImage = "${env.IMAGE_REPO}:${previousTag}"
        echo "Rolling back to: ${previousImage}"

        withCredentials([
            file(credentialsId: env.GCP_SA_CRED, variable: 'GOOGLE_APPLICATION_CREDENTIALS'),
            string(credentialsId: "mongo-uri-${deployEnv}", variable: 'MONGO_URI')
        ]) {
            sh """
                gcloud auth activate-service-account --key-file=\$GOOGLE_APPLICATION_CREDENTIALS

                ansible-playbook ansible/deploy.yml \
                    -i ansible/inventory/${deployEnv}.ini \
                    -e "image=${previousImage}" \
                    -e "app_env=${deployEnv}" \
                    -e "mongo_uri=\$MONGO_URI" \
                    -e "app_name=${env.APP_NAME}"
            """
        }

        // Verify rollback health
        def healthUrl = getHealthUrl(deployEnv)
        sh """
            sleep 20
            STATUS=\$(curl -s -o /dev/null -w '%{http_code}' ${healthUrl})
            if [ "\$STATUS" != "200" ]; then
                echo "ROLLBACK HEALTH CHECK FAILED — manual intervention required!"
                exit 1
            fi
            echo "Rollback successful — service healthy at ${healthUrl}"
        """

    } catch (Exception e) {
        echo "Rollback attempt failed: ${e.getMessage()}"
        echo "MANUAL ROLLBACK REQUIRED for ${deployEnv}!"
    }
}

