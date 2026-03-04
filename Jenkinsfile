// =============================================================================
// Twenty CRM - Build from Source & Push to DOCR
// =============================================================================
// Triggered automatically on git push to the Twenty CRM source repo (fork).
// Clones source → Docker build → Push to DOCR → Update infra repo overlays.
//
// Jenkins Pipeline SCM: Your Twenty CRM fork (e.g., github.com/OFactors/twenty)
// Infra Repo:           github.com/OFactors/Digitalocean-Infra
//
// Triggers: GitHub webhook (auto) or manual build
// =============================================================================

pipeline {
    agent any

    triggers {
        githubPush()
    }

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: '',
            description: 'Custom image tag (leave empty to auto-generate from git: branch-shortsha)'
        )
        string(
            name: 'DOCKERFILE_PATH',
            defaultValue: 'packages/twenty-docker/twenty/Dockerfile',
            description: 'Path to Dockerfile inside the Twenty CRM source repo'
        )
        booleanParam(
            name: 'UPDATE_CLIENTS',
            defaultValue: true,
            description: 'Update all client overlays in the infra repo with new image tag'
        )
        string(
            name: 'INFRA_REPO_URL',
            defaultValue: 'https://github.com/OFactors/Digitalocean-Infra.git',
            description: 'Infrastructure repo URL containing k8s-manifests'
        )
        string(
            name: 'INFRA_BRANCH',
            defaultValue: 'main',
            description: 'Infra repo branch'
        )
    }

    environment {
        DOCR_REGISTRY = 'registry.digitalocean.com/twentycrmprod'
        DOCR_IMAGE    = "${DOCR_REGISTRY}/twenty"
    }

    options {
        timestamps()
        ansiColor('xterm')
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.GIT_BRANCH_NAME  = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                    env.GIT_COMMIT_MSG   = sh(script: 'git log -1 --pretty=%s', returnStdout: true).trim()

                    // Auto-generate tag: branch-sha (e.g., main-a1b2c3d)
                    if (params.IMAGE_TAG?.trim()) {
                        env.RESOLVED_TAG = params.IMAGE_TAG.trim()
                    } else {
                        def safeBranch = env.GIT_BRANCH_NAME.replaceAll('[^a-zA-Z0-9.-]', '-')
                        env.RESOLVED_TAG = "${safeBranch}-${env.GIT_COMMIT_SHORT}"
                    }

                    echo """
=== Build Info ===
Branch:     ${env.GIT_BRANCH_NAME}
Commit:     ${env.GIT_COMMIT_SHORT}
Message:    ${env.GIT_COMMIT_MSG}
Image Tag:  ${env.RESOLVED_TAG}
Dockerfile: ${params.DOCKERFILE_PATH}
"""
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    echo "=== Building Twenty CRM from source ==="
                    docker build \
                        -f ${params.DOCKERFILE_PATH} \
                        -t ${DOCR_IMAGE}:${env.RESOLVED_TAG} \
                        -t ${DOCR_IMAGE}:latest \
                        --build-arg BUILD_COMMIT=${env.GIT_COMMIT_SHORT} \
                        .

                    echo "=== Build complete ==="
                    docker images ${DOCR_IMAGE}
                """
            }
        }

        stage('Push to DOCR') {
            steps {
                withCredentials([
                    string(credentialsId: 'do-token', variable: 'DIGITALOCEAN_ACCESS_TOKEN')
                ]) {
                    sh """
                        doctl auth init --access-token \$DIGITALOCEAN_ACCESS_TOKEN
                        doctl registry login

                        docker push ${DOCR_IMAGE}:${env.RESOLVED_TAG}
                        docker push ${DOCR_IMAGE}:latest

                        echo "=== Pushed ==="
                        echo "${DOCR_IMAGE}:${env.RESOLVED_TAG}"
                        echo "${DOCR_IMAGE}:latest"
                    """
                }
            }
        }

        stage('Update Client Overlays') {
            when {
                expression { params.UPDATE_CLIENTS == true }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-pat',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh """
                        echo "=== Cloning infra repo ==="
                        INFRA_URL=\$(echo "${params.INFRA_REPO_URL}" | sed "s|https://|https://\${GIT_USER}:\${GIT_TOKEN}@|")
                        rm -rf _infra_repo
                        git clone --branch ${params.INFRA_BRANCH} --depth 1 "\$INFRA_URL" _infra_repo

                        cd _infra_repo

                        echo "=== Updating client overlays to tag: ${env.RESOLVED_TAG} ==="

                        CLIENT_DIRS=\$(find k8s-manifests/clients/client-* -maxdepth 0 -type d 2>/dev/null || true)

                        if [ -z "\$CLIENT_DIRS" ]; then
                            echo "No client directories found, skipping."
                            exit 0
                        fi

                        for dir in \$CLIENT_DIRS; do
                            KUSTOMIZATION="\$dir/kustomization.yaml"
                            if [ -f "\$KUSTOMIZATION" ]; then
                                echo "Updating \$KUSTOMIZATION"
                                sed -i "s|newTag:.*|newTag: ${env.RESOLVED_TAG}|g" "\$KUSTOMIZATION"
                            fi
                        done

                        echo "=== Changes ==="
                        git diff --stat

                        if git diff --quiet; then
                            echo "No changes to commit."
                            exit 0
                        fi

                        git config user.email "jenkins@rupeequicker.com"
                        git config user.name "Jenkins CI"
                        git add k8s-manifests/clients/*/kustomization.yaml
                        git commit -m "chore: update Twenty CRM image to ${env.RESOLVED_TAG}

Source commit: ${env.GIT_COMMIT_SHORT} - ${env.GIT_COMMIT_MSG}"

                        git push origin ${params.INFRA_BRANCH}
                        echo "=== Committed and pushed to infra repo ==="
                    """
                }
            }
        }

        stage('Verify DOCR Image') {
            steps {
                withCredentials([
                    string(credentialsId: 'do-token', variable: 'DIGITALOCEAN_ACCESS_TOKEN')
                ]) {
                    sh """
                        doctl auth init --access-token \$DIGITALOCEAN_ACCESS_TOKEN
                        echo "=== DOCR Repository Tags ==="
                        doctl registry repository list-tags twenty --format Tag,CompressedSize,UpdatedAt
                    """
                }
            }
        }
    }

    post {
        success {
            echo """
============================================
  BUILD COMPLETE
============================================
Source:   ${env.GIT_BRANCH_NAME}@${env.GIT_COMMIT_SHORT}
Image:   ${DOCR_IMAGE}:${env.RESOLVED_TAG}
Clients: ${params.UPDATE_CLIENTS ? 'updated' : 'skipped'}

ArgoCD will auto-sync all client namespaces.
============================================
"""
        }
        failure {
            echo "Build FAILED. Check console output for errors."
        }
        always {
            sh """
                docker rmi ${DOCR_IMAGE}:${env.RESOLVED_TAG} || true
                docker rmi ${DOCR_IMAGE}:latest || true
                rm -rf _infra_repo || true
            """
        }
    }
}
