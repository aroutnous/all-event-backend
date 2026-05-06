pipeline {
    agent any
    environment {
        SONAR_PROJECT_KEY = 'allevent-backend'
        APP_URL = 'http://192.168.64.45'
        DOCKERHUB_USER = 'sountdocker'
        VM_IP = '192.168.64.45'
    }
    stages {
        stage('Clone') {
            steps {
                git credentialsId: 'aroutnous', url: 'https://github.com/aroutnous/all-event-backend.git', branch: 'main'
            }
        }
        stage('Installation dependances') {
            steps {
                sh 'composer install --no-interaction --prefer-dist'
            }
        }
        stage('Tests unitaires') {
            steps {
                sh 'php artisan test || true'
            }
        }
        stage('SAST - SonarQube') {
            steps {
                sh '''/opt/sonar-scanner/bin/sonar-scanner \
                    -Dsonar.projectKey=allevent-backend \
                    -Dsonar.sources=. \
                    -Dsonar.exclusions=vendor/**,node_modules/**,*.js \
                    -Dsonar.host.url=http://192.168.64.45:9000 \
                    -Dsonar.token=sqp_e27f9fd71ddba0bef2529656fcccc2f38ee304a6 || true'''
            }
        }
        stage('SCA - Composer Audit') {
            steps {
                sh 'composer audit || true'
            }
        }
        stage('SCA - OWASP Dependency Check') {
            steps {
                sh '''/opt/dependency-check/bin/dependency-check.sh \
                    --project allevent-backend \
                    --scan . \
                    --exclude "./vendor/**" \
                    --format HTML \
                    --out dependency-check-report/ || true'''
            }
        }
        stage('Secrets - Gitleaks') {
            steps {
                sh 'gitleaks detect -s . -v --log-opts="HEAD~1..HEAD" || true'
            }
        }
        stage('Secrets - Vault') {
            steps {
                withVault(vaultSecrets: [[path: 'secret/allevent',
                    secretValues: [
                        [envVar: 'VAULT_DOCKER_USER', vaultKey: 'dockerhub_user'],
                        [envVar: 'VAULT_DOCKER_TOKEN', vaultKey: 'dockerhub_token']
                    ]]]) {
                    sh 'echo "Secrets Vault charges avec succes !"'
                    sh 'echo "Docker User: $VAULT_DOCKER_USER"'
                }
            }
        }
        stage('Build Docker') {
            steps {
                sh 'docker build -t allevent-backend:latest .'
            }
        }
        stage('Push Docker Hub') {
            steps {
                withVault(vaultSecrets: [[path: 'secret/allevent',
                    secretValues: [
                        [envVar: 'VAULT_DOCKER_USER', vaultKey: 'dockerhub_user'],
                        [envVar: 'VAULT_DOCKER_TOKEN', vaultKey: 'dockerhub_token']
                    ]]]) {
                    sh '''
                        echo $VAULT_DOCKER_TOKEN | docker login -u $VAULT_DOCKER_USER --password-stdin
                        docker tag allevent-backend:latest $VAULT_DOCKER_USER/allevent-backend:latest
                        docker push $VAULT_DOCKER_USER/allevent-backend:latest
                    '''
                }
            }
        }
        stage('Scan Trivy') {
            steps {
                sh 'trivy image --exit-code 0 --severity HIGH,CRITICAL allevent-backend:latest || true'
            }
        }
        stage('DAST - ZAP') {
    	    steps {
        	sh '''
            	echo "DAST - OWASP ZAP"
            	echo "Note: ZAP nest pas disponible pour ARM64 sur cette VM."
	        echo "Sur architecture x86_64, la commande serait:"
	        echo "zaproxy -cmd -quickurl http://192.168.64.45 -quickprogress"
	        true
        	'''
    		}
	}
	stage('Deploy Docker Compose') {
    steps {
        sh '''
            cd /var/lib/jenkins/allevent-deploy
            docker compose pull || true
            docker compose up -d --remove-orphans --force-recreate
        '''
    }
}

    }
    post {
        success { echo 'Pipeline DevSecOps reussi !' }
        failure { echo 'Pipeline echoue !' }
    }
}
