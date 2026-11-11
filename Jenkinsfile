// Jenkins Scripted Pipeline for lern-service (AWS)
// Parameters (BRANCH, TAG) are defined in the job configuration.

node {
    // Names must match what is configured in Jenkins
    def mvnHome     = tool 'maven3'             // Manage Jenkins > Tools > Maven
    def imageRepo   = 'aastardev1/lern-service'
    def imageTag    = ''

    env.PATH = "${mvnHome}/bin:${env.PATH}"

    try {
        stage('Checkout') {
            cleanWs()
            checkout scm
            // Or, for a fixed repo instead of the job's SCM:
            // git branch: params.BRANCH, url: 'https://github.com/your-org/your-repo.git', credentialsId: 'github-creds'

            def shortCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
            imageTag = params.TAG?.trim() ? params.TAG.trim() : "${env.BUILD_NUMBER}-${shortCommit}"
            echo "Image tag: ${imageTag}"
        }

        stage('Maven Build') {
            sh 'mvn clean install -P lern,aws -DskipTests'
        }

        stage('Play2 Dist') {
            sh 'mvn play2:dist -pl modules/lern/service -P lern,aws'
        }

        stage('Docker Build') {
            sh "docker build -f build/lern/Dockerfile --build-arg CSP=aws -t ${imageRepo}:${imageTag} ."
        }

        stage('Docker Push') {
            withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                              usernameVariable: 'DOCKER_USER',
                                              passwordVariable: 'DOCKER_PASS')]) {
                sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                '''
                sh "docker push ${imageRepo}:${imageTag}"
            }
        }

        currentBuild.result = 'SUCCESS'
        currentBuild.description = "${imageRepo}:${imageTag}"

    } catch (err) {
        currentBuild.result = 'FAILURE'
        echo "Build failed: ${err}"
        throw err
    } finally {
        stage('Cleanup') {
            sh "docker rmi ${imageRepo}:${imageTag} || true"
            sh 'docker logout || true'
        }
    }
}
