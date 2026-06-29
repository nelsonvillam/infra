pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        AWS_REGION = 'sa-east-1'
        NAMESPACE  = 'shop'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install ESO') {
            steps {
                sh """
                    kubectl config use-context docker-desktop

                    if ! helm status external-secrets -n external-secrets > /dev/null 2>&1; then
                        helm repo add external-secrets https://charts.external-secrets.io 2>/dev/null || true
                        helm repo update
                        helm install external-secrets external-secrets/external-secrets \
                            -n external-secrets --create-namespace \
                            --set installCRDs=true \
                            --wait --timeout 120s
                    else
                        echo "ESO already installed, skipping"
                    fi
                """
            }
        }

        stage('Install nginx Ingress') {
            steps {
                sh """
                    if ! kubectl get deployment ingress-nginx-controller -n ingress-nginx > /dev/null 2>&1; then
                        kubectl apply -f \
                            https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
                        kubectl wait --namespace ingress-nginx \
                            --for=condition=ready pod \
                            --selector=app.kubernetes.io/component=controller \
                            --timeout=120s
                    else
                        echo "nginx Ingress already installed, skipping"
                    fi
                """
            }
        }

        stage('Apply Manifests') {
            steps {
                sh "kubectl apply -k k8s/overlays/local/"
            }
        }

        stage('Sync Secrets') {
            steps {
                withCredentials([
                    string(credentialsId: 'aws-access-key-id',     variable: 'CI_AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'CI_AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh """
                        set +x
                        kubectl create secret generic aws-credentials \
                            --namespace ${NAMESPACE} \
                            --from-literal=access-key-id="\${CI_AWS_ACCESS_KEY_ID}" \
                            --from-literal=secret-access-key="\${CI_AWS_SECRET_ACCESS_KEY}" \
                            --from-literal=session-token="" \
                            --dry-run=client -o yaml | kubectl replace --force -f -
                        set -x

                        kubectl annotate secretstore aws-secretsmanager \
                            --namespace ${NAMESPACE} \
                            force-sync=\$(date +%s) \
                            --overwrite

                        kubectl wait secretstore/aws-secretsmanager \
                            --namespace ${NAMESPACE} \
                            --for=condition=Ready \
                            --timeout=60s

                        for es in mongodb-credentials mongodb-keyfile shop-secret; do
                            kubectl annotate externalsecret/\$es \
                                --namespace ${NAMESPACE} \
                                force-sync=\$(date +%s) \
                                --overwrite
                        done

                        for es in mongodb-credentials mongodb-keyfile shop-secret; do
                            kubectl wait externalsecret/\$es \
                                --namespace ${NAMESPACE} \
                                --for=condition=Ready \
                                --timeout=120s
                            echo "\$es synced"
                        done
                    """
                }
            }
        }

        stage('Init MongoDB') {
            steps {
                sh """
                    kubectl wait --for=condition=ready pod/mongo-0 \
                        --namespace ${NAMESPACE} --timeout=180s

                    kubectl delete job mongodb-rs-init \
                        --namespace ${NAMESPACE} --ignore-not-found=true

                    kubectl apply -f k8s/base/mongodb/rs-init-job.yaml

                    kubectl wait --for=condition=complete job/mongodb-rs-init \
                        --namespace ${NAMESPACE} --timeout=120s
                """
            }
        }
    }

    post {
        success {
            echo "Infra deployed. Run gateway, shop, and ping-service pipelines next."
        }
        failure {
            echo "Infra pipeline failed. Check the logs above."
        }
    }
}
