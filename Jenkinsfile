pipeline {
    agent any
    
    environment {
        SONAR_HOST_URL = 'http://192.168.193.128:9000'
        SONAR_TOKEN = 'squ_b6986d26e43b9a5c9fe5c8247fb84abdb22995a4'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                script {
                    echo "📥 Cloning repository: main"
                }
                git branch: 'vente_voitures_1', url: 'https://github.com/motrabelsi10/testpipeline.git'
            }
        }
        
        stage('Export & Upload JWA to Nexus (joget-repo)') {
            steps {
                script {
                    echo "📦 Exporting application RSU from Joget..."
                }
                sh '''
                APP_NAME="vente_voitures"
                APP_ID="1"
                TIMESTAMP=$(date +"%Y%m%d%H%M%S")
                JWA_FILE="APP_${APP_NAME}-${APP_ID}-${TIMESTAMP}.jwa"
                EXPORT_PATH="/opt/joget/wflow/export"
                
                echo "📦 Exporting: Name=$APP_NAME, ID=$APP_ID, File=$JWA_FILE"
                
                docker exec -u root jogetapp bash -c "apt-get update && apt-get install -y zip"
                docker exec -u root jogetapp bash -c "mkdir -p ${EXPORT_PATH} && chmod -R 777 ${EXPORT_PATH}"
                docker exec jogetapp bash -c "cd /opt/joget/wflow/app_src && zip -r ${EXPORT_PATH}/${JWA_FILE} ${APP_NAME} -x \"${APP_NAME}/.git/*\""
                
                FILE_SIZE=$(docker exec jogetapp bash -c "stat -c %s ${EXPORT_PATH}/${JWA_FILE}")
                
                if [ "$FILE_SIZE" -lt 1024 ]; then
                    echo "❌ ERROR: Exported JWA file seems empty (size: ${FILE_SIZE} bytes)."
                    exit 1
                fi
                
                echo "✅ Export successful! File: ${JWA_FILE} (Size: ${FILE_SIZE} bytes)"
                
                docker cp jogetapp:${EXPORT_PATH}/${JWA_FILE} .
                
                if [ ! -f "${JWA_FILE}" ]; then
                    echo "❌ ERROR: JWA file was not copied correctly to Jenkins."
                    exit 1
                fi
                
                echo "📤 Uploading ${JWA_FILE} to Nexus..."
                
                curl -u admin:Mouhamed2000** --upload-file ${JWA_FILE} \
                "http://192.168.193.128:8082/repository/vente_voitures-repo/com/voitures/jwa/${APP_ID}/${JWA_FILE}"
                
                echo "✅ Upload completed successfully to joget-repo!"
                '''
            }
        }
        
        stage('Prepare SonarQube Scanner') {
            steps {
                script {
                    echo "🔍 Checking if SonarQube Scanner is installed..."
                }
                sh '''
                    if [ ! -d "sonar-scanner" ]; then
                        echo "🚀 Installing SonarQube Scanner..."
                        apt-get update && apt-get install -y wget unzip
                        wget -q https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip -O sonar-scanner.zip
                        unzip -q sonar-scanner.zip
                        mv sonar-scanner-5.0.1.3006-linux sonar-scanner
                        chmod +x sonar-scanner/bin/sonar-scanner
                    else
                        echo "✅ SonarQube Scanner is already installed."
                    fi
                '''
            }
        }
        
        stage('Prepare Joget Environment') {
            steps {
                sh 'docker exec -i jogetapp bash -c "mkdir -p /opt/joget/wflow/scripts/output && chmod -R 777 /opt/joget/wflow/scripts/output"'
            }
        }
        
        stage('Run Python Script') {
            steps {
                sh '''
                    docker exec -i jogetapp bash -c "python3 /opt/joget/wflow/scripts/extract_code.py"
                    docker exec -i jogetapp bash -c "python3 /opt/joget/wflow/scripts/extract_js.py"
                '''
            }
        }
        
        stage('Copy Extracted Files to Jenkins') {
            steps {
                sh '''
                    rm -rf output
                    mkdir -p output/java output/sql output/js
                    docker cp jogetapp:/opt/joget/wflow/scripts/output/java ./output/
                    docker cp jogetapp:/opt/joget/wflow/scripts/output/sql ./output/
                    docker cp jogetapp:/opt/joget/wflow/scripts/output/js ./output/
                '''
            }
        }
        
        stage('Verify Extracted Files in Jenkins') {
            steps {
                sh 'ls -R ./output || echo "🚨 No extracted files found!"'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                sh '''
                    if [ -d "./output/java" ] || [ -d "./output/sql" ] || [ -d "./output/js" ]; then
                        echo "✅ Files found, proceeding with SonarQube analysis..."
                        # Exécuter l'analyse SonarQube
                        sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=voitures \
                          -Dsonar.projectName=voitures \
                          -Dsonar.host.url="${SONAR_HOST_URL}" \
                          -Dsonar.login="${SONAR_TOKEN}" \
                          -Dsonar.sources=./output \
                          -Dsonar.inclusions="**/*.java,**/*.sql,**/*.js" \
                          -Dsonar.java.binaries=. \
                          -Dsonar.dependencyCheck.reportPath=./output/reports/dependency-check-report.json \
                          -Dsonar.dependencyCheck.htmlReportPath=./output/reports/dependency-check-report.html \
                          -Dsonar.scm.provider=git \
                          -Dsonar.issue.ignore.multicriteria=e1 \
                          -Dsonar.issue.ignore.multicriteria.e1.ruleKey=squid:S00117 \
                          -Dsonar.issue.ignore.multicriteria.e1.resourceKey=**/*.sql \
                          -Dsonar.sourceEncoding=UTF-8 \
                          -Dsonar.verbose=true
                    else
                        echo "🚨 ERROR: No Java, SQL, or JS files found in ./output!"
                        exit 1
                    fi
                '''
            }
        }

        
    }
        
        post {
            success {
                script {
                    mail to: 'mouhamedtrabelsi.28@gmail.com',
                         subject: "Succès du build: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                         body: "Le build a réussi ! Consultez le lien suivant : ${env.BUILD_URL}"
                }
            }
            failure {
                script {
                    mail to: 'mouhamedtrabelsi.28@gmail.com',
                         subject: "Échec du build: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                         body: "Le build a échoué ! Consultez le lien suivant : ${env.BUILD_URL}"
                }
            }
        }
    
}
