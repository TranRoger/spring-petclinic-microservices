def changeFiles = []
def changedServices = []
def allServices = true

pipeline {
    agent any

    tools {
        maven 'M3'
        jdk 'jdk17'
    }

    stages {
        stage('Checkout & Initialize') {
            steps {
                script {
                    // Full depth checkout
                    checkout([
                        $class: 'GitSCM',
                        branches: scm.branches,
                        userRemoteConfigs: scm.userRemoteConfigs,
                        extensions: [
                            [$class: 'CloneOption', shallow: false, depth: 0]
                        ]
                    ])

                    def targetBranch = env.CHANGE_TARGET ?: env.BRANCH_NAME
                    sh "git fetch origin ${targetBranch}"

                    def diffOutput = []

                    if (env.CHANGE_TARGET) {
                        // Pull request build using merge strategy
                        echo "Detected PR build to '${env.CHANGE_TARGET}'. Comparing HEAD~2 (PR head) with HEAD (merged result)"
                        diffOutput = sh(script: "git diff --name-only HEAD~2 HEAD", returnStdout: true).trim()
                    } else {
                        // Normal push to a branch
                        echo "Detected branch push to '${env.BRANCH_NAME}. Comparing previous commit (HEAD~1) with current (HEAD)"
                        diffOutput = sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim()
                    }
                    changeFiles = diffOutput ? diffOutput.split("\n").collect { it.trim() } : []

                    echo "Changed files: \n${changeFiles.join('\n')}"

                    changedServices = getChangedServices(changeFiles)
                    echo "Changed services:\n${changedServices.join('\n')}"

                    if (changedServices.isEmpty()) {
                        skipBuild = true
                        currentBuild.result = 'NOT_BUILT'
                        echo "No changes detected - skipping build and test stages"
                    }

                    allServices = isFullBuild(changedServices)
                }
            }
        }

        stage('Build services') {
            when {
                expression { return !changedServices.isEmpty() }
            }
            steps {
                script {
                    if (allServices) {
                        echo "Full build triggered"
                        sh './mvnw clean package -DskipTests'
                    } else {
                        changedServices.each { service ->
                            echo "Building ${serivce}"
                            sh "./mvnw clean package -DskipTests -pl ${service} -am"
                        }
                    }
                }
            }
        }

        stage('Test Services') {
            when {
                expression { return !changedServices.isEmpty() }
            }
            steps {
                script {
                    if (allServices) {
                        echo "Testing all servicees"
                        sh './mvnw test'
                        junit '**/target/surefire-reports/*.xml'
                    } else {
                        changedServices.each { service -> 
                            echo "Testing ${service}"
                            sh "./mvnw test -pl :${service}"
                            junit "**/${service}/target/surefire-reports/*.xml"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                if (!changedServices.isEmpty()) {
                    junit testResults: '**/target/surefire-reports/*xml', allowEmptyResults: true
                    
                    def coveragePattern = allServices ?
                        '**/target/site/jacoco/jacoco.xml' :
                        changedServices.collect {
                            "**/${it}/target/site/jacoco/jacoco.xml"
                        }.join(',')

                    recordCoverage(
                        tools: [[parser: 'JACOCO', pattern: coveragePattern]],
                        
                        // look for sources in any sub-dir that contains src/main/java

                        sourceDirectories : [[path: 'glob:**/src/main/java']],
                        sourceCodeRetention: 'MODIFIED',    // or NEVER | LAST_BUILD | EVERY_BUILD

                        qualityGates: [[
                            metric      :   'LINE',
                            baseline    :   'PROJECT',
                            threshold   :   70,
                            criticality :   'FAILURE'       // fail build if < 70 % coverage
                        ]]
                    )
                }
            }
        }
    }
}

// Return List<String> of changed services
def getChangedServices(List changes) {
    def serviceMap = [
        'spring-petclinic-api-gateway'      : 'spring-petclinic-api-gateway',
        'spring-petclinic-customers-service': 'spring-petclinic-customers-service',
        'spring-petclinic-vets-service'     : 'spring-petclinic-vets-service',
        'spring-petclinic-visits-service'   : 'spring-petclinic-visits-service',
        'spring-petclinic-config-server'    : 'spring-petclinic-config-server',
        'spring-petclinic-discovery-server' : 'spring-petclinic-discovery-server',
        'spring-petclinic-admin-server'     : 'spring-petclinic-admin-server'
    ]

    def services = []
    changes.each { change -> 
        serviceMap.each { dir, service ->
            if (change.contains(dir) && !services.contains(service)) {
                services << service
            }
        }
    }

    if (changes.any { it.contains('pom.xml') || it.contains('Jenkinsfile') }) {
        return ['ALL'] // trigger full build
    }

    return services

}

// Helper function to detect full build flag
def isFullBuild(List services) {
    return services.size() == 1 && services[0] == 'ALL'
}