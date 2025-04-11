pipeline {
    agent any
    
    tools {
        maven 'M3'
        jdk 'jdk17'
    }

    environment {
        COVERAGE_THRESHOLD = 70
        GITHUB_REPO = 'https://github.com/TranRoger/spring-petclinic-microservices.git'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.GIT_BRANCH}"]],
                    extensions: [[$class: 'CleanBeforeCheckout']],
                    userRemoteConfigs: [[
                        url: env.GITHUB_REPO,
                        credentialsId: 'github-token'
                    ]]
                ])
            }
        }

        stage('Build') {
            steps {
                script {
                    def changedServices = getChangedServices()
                    
                    if (changedServices.isEmpty()) {
                        echo "Building all services"
                        sh 'mvn clean package -DskipTests'
                    } else {
                        changedServices.each { service ->
                            echo "Building ${service}"
                            sh "mvn -pl spring-petclinic-${service} clean package -DskipTests"
                        }
                    }
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    def changedServices = getChangedServices()
                    
                    if (changedServices.isEmpty()) {
                        echo "Testing all services"
                        sh 'mvn test'
                    } else {
                        changedServices.each { service ->
                            echo "Testing ${service}"
                            sh "mvn -pl spring-petclinic-${service} test"
                        }
                    }
                }
            }
            
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    // Generate coverage reports in Cobertura format
                    sh 'mvn cobertura:cobertura -Dcobertura.report.format=xml'
                }
            }
        }

        stage('Coverage Report') {
            steps {
                script {
                    publishCoverage(
                        adapters: [
                            coberturaAdapter(path: '**/target/site/cobertura/coverage.xml')
                        ],
                        sourceFileResolver: sourceFiles('NEVER_STORE')
                    )
                    
                    // Check coverage thresholds
                    def changedServices = getChangedServices()
                    if (changedServices.isEmpty()) {
                        changedServices = getAllServices()
                    }
                    
                    changedServices.each { service ->
                        checkServiceCoverage(service)
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}

// ============= Utility Functions =============

def getChangedServices() {
    def changes = currentBuild.changeSets.collectMany { it.items.collectMany { it.affectedFiles.collect { it.path } } }
    if (changes.isEmpty()) {
        return []
    }
    
    def serviceMap = [
        'spring-petclinic-discovery-server': 'discovery-server',
        'spring-petclinic-admin-server': 'admin-server',
        'spring-petclinic-customers-service': 'customers-service',
        'spring-petclinic-vets-service': 'vets-service',
        'spring-petclinic-visits-service': 'visits-service',
        'spring-petclinic-api-gateway': 'api-gateway'
    ]
    
    def services = [] as Set
    
    changes.each { file ->
        serviceMap.each { dir, service ->
            if (file.startsWith(dir)) {
                services.add(service)
            }
        }
        
        // Rebuild all if build config changes
        if (file.contains('pom.xml') || file.contains('Jenkinsfile')) {
            services.clear()
            services.add('all')
            return
        }
    }
    
    return services as List
}

def checkServiceCoverage(service) {
    if (service == 'all') return
    
    def coverageFile = "spring-petclinic-${service}/target/site/cobertura/coverage.xml"
    if (!fileExists(coverageFile)) {
        echo "No coverage report found for ${service}"
        return
    }
    
    def coverage = getCoberturaCoverage(coverageFile)
    echo "Coverage for ${service}: Line=${coverage.line}%, Branch=${coverage.branch}%"
    
    if (coverage.line < env.COVERAGE_THRESHOLD.toInteger()) {
        unstable "Line coverage for ${service} (${coverage.line}%) is below ${env.COVERAGE_THRESHOLD}% threshold"
    }
}

def getCoberturaCoverage(xmlFile) {
    def report = readFile xmlFile
    def metrics = new XmlSlurper().parseText(report).coverage[0].metrics[0]
    
    return [
        line: metrics.@'line-rate'.toDouble() * 100,
        branch: metrics.@'branch-rate'.toDouble() * 100
    ]
}

def getAllServices() {
    return [
        'discovery-server',
        'admin-server',
        'customers-service',
        'vets-service',
        'visits-service',
        'api-gateway'
    ]
}
