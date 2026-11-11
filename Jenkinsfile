// Jenkins Scripted Pipeline for lern-service (AWS)
// Parameter BRANCH is defined in the job configuration.
// Image tag = <branch>-<short commit>, e.g. main-a1b2c3d
//
// Prerequisite: the Jenkins agent must already be logged in to Docker Hub
// as the jenkins user (run once on the agent: sudo -u jenkins docker login -u aastardev1)

// Runs Maven inside a container instead of needing it on the agent
def mvn(String args) {
    sh """
        mkdir -p "\$WORKSPACE/.m2"
        docker run --rm \\
            -u "\$(id -u):\$(id -g)" \\
            -v "\$PWD":/src \\
            -v "\$WORKSPACE/.m2":/var/maven/.m2 \\
            -e MAVEN_CONFIG=/var/maven/.m2 \\
            -w /src \\
            maven:3.9-eclipse-temurin-11 \\
            mvn -B -Duser.home=/var/maven ${args}
    """
}

node {
    def imageRepo   = 'aastardev1/lern-service'
    def imageTag    = ''

    try {
        stage('Checkout') {
            // Clean everything except the Maven cache
            cleanWs(patterns: [[pattern: '.m2/**', type: 'EXCLUDE']])
            checkout scm
            // Or, for a fixed repo instead of the job's SCM:
            // git branch: params.BRANCH, url: 'https://github.com/your-org/your-repo.git', credentialsId: 'github-creds'

            def shortCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
            def branch      = (params.BRANCH ?: env.BRANCH_NAME ?: 'unknown').trim()
            // Docker tags can't contain '/', so feature/foo -> feature-foo
            def safeBranch  = branch.replaceAll('[^A-Za-z0-9_.-]', '-')
            imageTag = "${safeBranch}-${shortCommit}"
            echo "Image tag: ${imageTag}"
        }

        stage('Maven Build') {
            mvn('clean install -P lern,aws -DskipTests')
        }

        stage('Play2 Dist') {
            mvn('play2:dist -pl modules/lern/service -P lern,aws')
        }

        stage('Docker Build') {
            sh "docker build -f build/lern/Dockerfile --build-arg CSP=aws -t ${imageRepo}:${imageTag} ."
        }

        stage('Docker Push') {
            sh "docker push ${imageRepo}:${imageTag}"
        }

        currentBuild.result = 'SUCCESS'
        currentBuild.description = "${imageRepo}:${imageTag}"

    } catch (err) {
        currentBuild.result = 'FAILURE'
        echo "Build failed: ${err}"
        throw err
    } finally {
        stage('Cleanup') {
            // Remove the local image to save disk space; keep the agent's Docker Hub login
            sh "docker rmi ${imageRepo}:${imageTag} || true"
        }
    }
}
