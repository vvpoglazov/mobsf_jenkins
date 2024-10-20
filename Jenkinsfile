pipeline {
    agent any
    environment {
        MOBSF_API_KEY = 'Dummy'
    }
    stages {
        stage('Start MobSF Container') {
            steps {
                script {
                    // Запуск MobSF как отдельного контейнера в фоновом режиме с интерполяцией переменной API-ключа
                    sh "docker run -d -p 8000:8000 -e MOBSF_API_KEY=${MOBSF_API_KEY} --name mobsf-container opensecurity/mobile-security-framework-mobsf:latest"
                    
                    // Даем время на запуск MobSF
                    sleep time: 50, unit: 'SECONDS'
                    
                    // Получаем IP-адрес контейнера MobSF
                    def containerIp = sh (
                        script: "docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mobsf-container",
                        returnStdout: true
                    ).trim()
                    
                    // Устанавливаем URL для MobSF API с динамическим IP-адресом
                    env.MOBSF_API_URL = "http://172.18.0.2:8000/api/v1"
                    echo "MobSF API URL: ${env.MOBSF_API_URL}"
                    
                }
            }
        }
        stage('Upload APK') {
            steps {
                script {
                    try {
                        // Укажи путь к APK файлу, который был загружен в workspace Jenkins
                        def apkPath = "${env.WORKSPACE}/InsecureBankv2.apk"
                        
                        // Загружаем APK в MobSF для анализа и получаем JSON ответ
                        def uploadResponse = sh (
                            script: """
                                curl -s -F 'file=@${apkPath}' \
                                -F 'scan_type=apk' -F 're_scan=1' \
                                -H 'Authorization: ${MOBSF_API_KEY}' \
                                ${MOBSF_API_URL}/upload
                            """,
                            returnStdout: true
                        ).trim()
                        
                        echo "Upload Response: ${uploadResponse}"
                        
                        // Парсим JSON ответ с помощью JsonSlurper
                        def jsonSlurper = new groovy.json.JsonSlurper()
                        def parsedResponse = jsonSlurper.parseText(uploadResponse)
                        env.HASH = parsedResponse.hash
                        echo "APK Hash: ${env.HASH}"
                    } catch (Exception e) {
                        echo "Error during APK upload or response parsing: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage('Initiate Scan') {
            steps {
                script {
                    try {
                        // Инициализируем сканирование по хешу APK-файла
                        def scanResponse = sh (
                            script: """
                                curl -s -X POST ${MOBSF_API_URL}/scan \
                                -d "hash=${HASH}" \
                                -H "Authorization: ${MOBSF_API_KEY}"
                            """,
                            returnStdout: true
                        ).trim()
                        echo "Scan Completed"
                    } catch (Exception e) {
                        echo "Error during scan initiation: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage('Download PDF Report') {
            steps {
                script {
                    try {
                        // Скачиваем PDF отчет по хешу
                        sh """
                            curl -X POST ${MOBSF_API_URL}/download_pdf \
                            -d "hash=${HASH}" \
                            -H "Authorization: ${MOBSF_API_KEY}" \
                            --output ${env.WORKSPACE}/mobsf_report.pdf
                        """
                        echo "PDF report has been downloaded and saved to ${env.WORKSPACE}/mobsf_report.pdf."
                    } catch (Exception e) {
                        echo "Error downloading PDF report: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
    }
    post {
        always {
            // Останавливаем контейнер MobSF после завершения работы
            sh 'docker stop mobsf-container && docker rm mobsf-container'
        }
    }
}
