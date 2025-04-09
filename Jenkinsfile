pipeline {
    agent any

    environment {
        MOD_FILES = ''
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    def services = [] // Danh sách service thay đổi
                    MOD_FILES = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim()
                    echo "🔍 Modified files: ${MOD_FILES}"

                    MOD_FILES.split("\n").each { file ->
                        if (file.startsWith("spring-petclinic-") && file.split("/").size() > 1) {
                            def svc = file.split("/")[0]
                            if (!services.contains(svc)) {
                                services << svc
                            }
                        }
                    }

                    if (services.isEmpty()) {
                        echo "✅ No changes detected, skipping."
                        currentBuild.result = 'SUCCESS'
                        return
                    }

                    echo "⚙️ Affected services: ${services}"
                    env.SERVICES = services.join(',') // Lưu danh sách service thay đổi
                }
            }
        }

        stage('Test & Coverage') {
            when {
                expression { return env.SERVICES != null && env.SERVICES != "" }
            }
            steps {
                script {
                    def services = env.SERVICES.split(',')
                    services.each { svc ->
                        echo "🧪 Testing: ${svc}"
                        dir(svc) {
                            sh '../mvnw clean verify -PbuildDocker jacoco:report'
                            def jacocoFile = sh(script: "find target -name jacoco.xml", returnStdout: true).trim()

                            echo "1111"
                            if (!jacocoFile) {
                                echo "⚠️ No JaCoCo report found for ${service}."
                            } else {
                                def missed = sh(
                                    script: """awk -F'missed="' '/<counter type="LINE"/ {gsub(/".*/, "", \$2); sum += \$2} END {print sum}' ${jacocoFile}""",
                                    returnStdout: true
                                ).trim()

                                def covered = sh(
                                    script: """awk -F'covered="' '/<counter type="LINE"/ {gsub(/".*/, "", \$2); sum += \$2} END {print sum}' ${jacocoFile}""",
                                    returnStdout: true
                                ).trim()

                                echo "22222"

                                def total = missed.toInteger() + covered.toInteger()
                                def coveragePercent = (total > 0) ? (covered.toInteger() * 100 / total) : 0

                                echo "🚀 Test coverage for ${service}: ${coveragePercent}%"

                                if (coveragePercent < 70) {
                                    error("❌ Test coverage below 70% for ${service}.")
                                }
                            }
                        }
                    }
                }
            }
            // post {
            //     always {
            //         junit '**/target/surefire-reports/*.xml'
            //         script {
            //             def services = env.SERVICES.split(',')
            //             services.each { svc ->
            //                 echo "📊 Generating JaCoCo for: ${svc}"
            //                 jacoco(
            //                     execPattern: "${svc}/target/jacoco.exec",
            //                     classPattern: "${svc}/target/classes",
            //                     sourcePattern: "${svc}/src/main/java",
            //                     exclusionPattern: "${svc}/src/test/**",
            //                     minimumLineCoverage: '70',
            //                     changeBuildStatus: true
            //                 )
            //             }
            //         }
            //     }
            // }
        }

        // stage('Build') {
        //     when {
        //         expression { return env.SERVICES != null && env.SERVICES != "" }
        //     }
        //     steps {
        //         script {
        //             def services = env.SERVICES.split(',')
        //             services.each { svc ->
        //                 echo "🔨 Building: ${svc}"
        //                 dir(svc) {
        //                     sh '../mvnw clean package -DskipTests'
        //                 }
        //             }
        //         }
        //     }
        // }
    }

    post {
        success {
            script {
                def commitId = env.GIT_COMMIT
                echo "✅ Sending 'success' to GitHub: ${commitId}"
                def response = httpRequest(
                    url: "https://api.github.com/repos/ndmanh3003/spring-petclinic-microservices/statuses/${commitId}",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: """{
                        \"state\": \"success\",
                        \"description\": \"Build passed\",
                        \"context\": \"ci/jenkins-pipeline\",
                        \"target_url\": \"${env.BUILD_URL}\"
                    }""",
                    authentication: 'github-token'
                )
                echo "📡 GitHub Response: ${response.status}"
            }
        }

        failure {
            script {
                def commitId = env.GIT_COMMIT
                echo "❌ Sending 'failure' to GitHub: ${commitId}"
                def response = httpRequest(
                    url: "https://api.github.com/repos/ndmanh3003/spring-petclinic-microservices/statuses/${commitId}",
                    httpMode: 'POST',
                    contentType: 'APPLICATION_JSON',
                    requestBody: """{
                        \"state\": \"failure\",
                        \"description\": \"Build failed\",
                        \"context\": \"ci/jenkins-pipeline\",
                        \"target_url\": \"${env.BUILD_URL}\"
                    }""",
                    authentication: 'github-token'
                )
                echo "📡 GitHub Response: ${response.status}"
            }
        }

        always {
            echo "🔚 Pipeline execution complete."
        }
    }
}
