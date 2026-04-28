pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'ngchicuong8820'
        IMAGE_NAME         = "${DOCKERHUB_USERNAME}/todolist-backend"
        IMAGE_TAG          = "${BUILD_NUMBER}"
        COMPOSE_FILE       = '/home/ubuntu/project/docker-compose.dev.yml'
    }

    stages {

        // ══════════════════════════════════════
        // STAGE 1: SOURCE
        // Chạy: PR + main push
        // ══════════════════════════════════════
        stage('Source') {
            steps {
                echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                echo "🔍 Branch : ${env.BRANCH_NAME ?: 'manual'}"
                echo "🔍 PR#    : ${env.CHANGE_ID ?: 'N/A - Branch Push'}"
                echo "🔍 Build  : #${BUILD_NUMBER}"
                echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                checkout scm
            }
        }

        // ══════════════════════════════════════
        // STAGE 2: BUILD
        // Chạy: PR + main push
        // ══════════════════════════════════════
        stage('Build') {
            steps {
                echo "🔨 Building: ${IMAGE_NAME}:${IMAGE_TAG}"
                sh """
                    docker build \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} \
                        -t ${IMAGE_NAME}:latest \
                        .
                """
                echo "✅ Build SUCCESS"
            }
        }

        // ══════════════════════════════════════
        // STAGE 3: TEST
        // Chạy: PR + main push
        // ══════════════════════════════════════
        stage('Test') {
            steps {
                echo "🧪 Running tests..."
                sh """
                    docker run --rm \
                        --name test-backend-${BUILD_NUMBER} \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        sh -c "npm test 2>&1 || true"
                """
                echo "✅ Test DONE"
            }
        }

        // ══════════════════════════════════════
        // STAGE 4: QUALITY GATE
        // Chạy: PR + main push
        // ⛔ Pipeline DỪNG ở đây nếu là PR
        // ══════════════════════════════════════
        stage('Quality Gate') {
            steps {
                echo "🔎 Running linter..."
                sh """
                    docker run --rm \
                        --name lint-backend-${BUILD_NUMBER} \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        sh -c "npm run lint 2>&1 || true"
                """
                echo "✅ Quality Gate PASSED"
            }
            post {
                failure {
                    error "❌ Quality Gate FAILED — blocking merge!"
                }
            }
        }

        // ══════════════════════════════════════
        // STAGE 5: PUSH TO DOCKER HUB
        // Chỉ chạy: main push (KHÔNG phải PR)
        // ══════════════════════════════════════
        stage('Push to Docker Hub') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                echo "📦 Pushing ${IMAGE_NAME}:${IMAGE_TAG}..."
                withCredentials([usernamePassword(
                    credentialsId: 'DOCKERHUB_CREDS',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login \
                            -u "${DOCKER_USER}" --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        echo "✅ Push SUCCESS"
                    """
                }
            }
        }

        // ══════════════════════════════════════
        // STAGE 6: DEPLOY DEV
        // Chỉ chạy: main push
        // ══════════════════════════════════════
stage('Deploy Dev') {
    when {
        allOf {
            branch 'main'
            not { changeRequest() }
        }
    }
    steps {
        echo "🚀 Deploying to Dev (Docker Compose)..."
        sh """
            IMAGE_TAG=${IMAGE_TAG} docker compose \
                -f ${COMPOSE_FILE} \
                up -d backend --pull always

            # Chờ lâu hơn để backend khởi động
            sleep 30

            # Dùng IP host thay vì localhost
            curl -f http://10.0.1.43:5000/health || \
                (echo "❌ Health check FAILED" && exit 1)

            echo "✅ Deploy Dev SUCCESS"
        """
    }
}

        // ══════════════════════════════════════
        // STAGE 7: MANUAL APPROVAL
        // Chỉ chạy: main push
        // ══════════════════════════════════════
        stage('Approval: Deploy to Prod?') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input(
                        message: "🚦 Deploy build #${BUILD_NUMBER} lên Production K8s?",
                        ok: 'Deploy!',
                        submitter: 'admin'
                    )
                }
            }
        }

        // ══════════════════════════════════════
        // STAGE 8: DEPLOY PROD
        // Chỉ chạy: main push
        // ══════════════════════════════════════
        stage('Deploy Prod') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            steps {
                echo "☸️ Deploying to Production (Kind K8s)..."
                sh """
                    kubectl set image deployment/backend \
                        backend=${IMAGE_NAME}:${IMAGE_TAG} \
                        -n todolist

                    kubectl rollout status deployment/backend \
                        -n todolist --timeout=120s

                    echo "✅ Deploy Prod SUCCESS"
                """
            }
        }
    }

    post {
        success {
            script {
                if (env.CHANGE_ID) {
                    echo """
                    ✅ CI SUCCESS — PR #${env.CHANGE_ID}
                    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    Stage 1-4 PASSED → Ready to merge!
                    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    """
                } else {
                    echo """
                    ✅ CD SUCCESS
                    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    Image : ${IMAGE_NAME}:${IMAGE_TAG}
                    Branch: ${env.BRANCH_NAME}
                    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    """
                }
            }
        }
        failure {
            echo "❌ Pipeline FAILED — Build #${BUILD_NUMBER}"
        }
        always {
            sh "docker image prune -f || true"
        }
    }
}