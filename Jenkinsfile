pipeline {
    agent any
    
    tools {
        maven 'M3'
        jdk 'jdk17'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                script {
                    // Safe way to get changed files without script approval
                    def changedFiles = getChangedFilesSafe()
                    def services = changedFiles ? getChangedServicesSafe(changedFiles) : getAllServicesSafe()
                    
                    if (services.contains('all')) {
                        sh 'mvn clean package'
                    } else {
                        services.each { service ->
                            sh "mvn -pl spring-petclinic-${service} clean package"
                        }
                    }
                }
            }
        }

        stage('Coverage') {
            steps {
                sh 'mvn cobertura:cobertura'
                publishCoverage(
                    adapters: [coberturaAdapter('**/target/site/cobertura/coverage.xml')],
                    sourceFileResolver: sourceFiles('NEVER_STORE')
                )
            }
        }
    }
}

// Approved-safe utility methods
@NonCPS
def getChangedFilesSafe() {
    return sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true)
        .trim()
        .split('\n')
        .toList()
}

@NonCPS
def getChangedServicesSafe(changedFiles) {
    def services = []
    changedFiles.each { file ->
        if (file.contains('spring-petclinic-discovery-server/')) services << 'discovery-server'
        if (file.contains('spring-petclinic-admin-server/')) services << 'admin-server'
        if (file.contains('spring-petclinic-customers-service/')) services << 'customers-service'
        if (file.contains('spring-petclinic-vets-service/')) services << 'vets-service'
        if (file.contains('spring-petclinic-visits-service/')) services << 'visits-service'
        if (file.contains('spring-petclinic-api-gateway/')) services << 'api-gateway'
        if (file.contains('pom.xml') || file.contains('Jenkinsfile')) return ['all']
    }
    return services.unique()
}

@NonCPS
def getAllServicesSafe() {
    return [
        'discovery-server',
        'admin-server',
        'customers-service',
        'vets-service',
        'visits-service',
        'api-gateway'
    ]
}
