pipeline {
    tools {
        maven 'M3'
        jdk 'jdk17' // Make sure this matches your Jenkins JDK configuration
    }

    environment {
        GIT_CREDENTIALS = credentials('github-token') // Configure this in Jenkins credentials
        COVERAGE_THRESHOLD = 70
        SLACK_CHANNEL = '#your-slack-channel' // Optional
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout & Initialize') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.GIT_BRANCH}"]],
                    extensions: [
                        [$class: 'CloneOption', depth: 1, noTags: false, shallow: true],
                        [$class: 'CleanBeforeCheckout']
                    ],
                    userRemoteConfigs: [[
                        credentialsId: 'github-token',
                        url: 'https://github.com/your-username/spring-petclinic-microservices.git'
                    ]]
                ])
                
                sh 'mvn --version'
                sh 'java -version'
            }
        }

        stage('Determine Changed Services') {
            steps {
                script {
                    // Get list of changed files
                    def changedFiles = getChangedFiles()
                    
                    // Map to services
                    def services = determineChangedServices(changedFiles)
                    
                    if (services.isEmpty()) {
                        echo "No service-specific changes detected. Will build all services."
                        services = getAllServices()
                    }
                    
                    // Store in environment variable for later stages
                    env.CHANGED_SERVICES = services.join(',')
                }
            }
        }

        stage('Build Services') {
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(',') as List
                    
                    if (services.contains('all')) {
                        echo "Building all services"
                        sh 'mvn clean package -DskipTests'
                    } else {
                        services.each { service ->
                            echo "Building service: ${service}"
                            sh "mvn -pl spring-petclinic-${service} clean package -DskipTests"
                        }
                    }
                }
            }
        }

        stage('Test Services') {
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(',') as List
                    
                    if (services.contains('all')) {
                        echo "Testing all services"
                        sh 'mvn test'
                    } else {
                        services.each { service ->
                            echo "Testing service: ${service}"
                            sh "mvn -pl spring-petclinic-${service} test"
                        }
                    }
                }
            }
            
            post {
                always {
                    junit allowEmptyResults: true, 
                          testResults: '**/target/surefire-reports/*.xml'
                    jacoco execPattern: '**/target/jacoco.exec',
                           classPattern: '**/target/classes',
                           sourcePattern: '**/src/main/java',
                           exclusionPattern: '**/generated/**'
                }
            }
        }

        stage('Verify Coverage') {
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(',') as List
                    
                    if (services.contains('all')) {
                        services = getAllServices()
                    }
                    
                    services.each { service ->
                        verifyServiceCoverage(service)
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            when {
                branch 'main' // Only run on main branch
            }
            steps {
                script {
                    withSonarQubeEnv('sonarqube') { // Configure in Jenkins
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }
    }

    post {
        success {
            slackSend(color: 'good', 
                     message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.GIT_BRANCH})")
        }
        failure {
            slackSend(color: 'danger', 
                     message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.GIT_BRANCH})")
            emailext(
                subject: "FAILED: Job '${env.JOB_NAME}' (${env.BUILD_NUMBER})",
                body: """Check console output at ${env.BUILD_URL}""",
                to: 'team@yourdomain.com'
            )
        }
        unstable {
            slackSend(color: 'warning', 
                     message: "UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.GIT_BRANCH})")
        }
        cleanup {
            cleanWs() // Clean workspace after build
        }
    }
}

// ============= Utility Functions =============

def getChangedFiles() {
    def changes = []
    if (env.CHANGE_ID) { // For PR builds
        def changeLogSets = currentBuild.changeSets
        for (int i = 0; i < changeLogSets.size(); i++) {
            def entries = changeLogSets[i].items
            for (int j = 0; j < entries.length; j++) {
                def entry = entries[j]
                def files = new ArrayList(entry.affectedFiles)
                for (int k = 0; k < files.size(); k++) {
                    changes.add(files[k].path)
                }
            }
        }
    } else { // For branch builds
        def diffCmd = "git diff --name-only HEAD~1 HEAD"
        changes = sh(script: diffCmd, returnStdout: true).split('\n')
    }
    return changes.findAll { it } // Remove empty entries
}

def determineChangedServices(changedFiles) {
    def serviceMap = [
        'spring-petclinic-discovery-server': 'discovery-server',
        'spring-petclinic-admin-server': 'admin-server',
        'spring-petclinic-customers-service': 'customers-service',
        'spring-petclinic-vets-service': 'vets-service',
        'spring-petclinic-visits-service': 'visits-service',
        'spring-petclinic-api-gateway': 'api-gateway'
    ]
    
    def services = [] as Set
    
    changedFiles.each { file ->
        serviceMap.each { dir, service ->
            if (file.startsWith(dir)) {
                services.add(service)
            }
        }
        
        // If build config changes, rebuild all
        if (file.contains('pom.xml') || file.contains('Jenkinsfile')) {
            services.clear()
            services.add('all')
            return // break out of loop
        }
    }
    
    return services as List
}

def verifyServiceCoverage(service) {
    def coverageFile = "spring-petclinic-${service}/target/site/jacoco/jacoco.csv"
    
    if (!fileExists(coverageFile)) {
        error "No coverage report found for ${service}"
    }
    
    def coverage = calculateCoverage(coverageFile)
    echo "Coverage for ${service}: ${coverage}% (Threshold: ${env.COVERAGE_THRESHOLD}%)"
    
    if (coverage < env.COVERAGE_THRESHOLD.toInteger()) {
        unstable "Code coverage for ${service} (${coverage}%) is below required ${env.COVERAGE_THRESHOLD}%"
    }
}

def calculateCoverage(reportPath) {
    def report = readFile reportPath
    def lines = report.split('\n')
    def missed = 0
    def covered = 0
    
    // Skip header line
    for (int i = 1; i < lines.size(); i++) {
        def cols = lines[i].split(',')
        missed += cols[3].toInteger() // INSTRUCTION_MISSED
        covered += cols[4].toInteger() // INSTRUCTION_COVERED
    }
    
    def total = missed + covered
    return total > 0 ? (covered * 100 / total).round(2) : 0
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
