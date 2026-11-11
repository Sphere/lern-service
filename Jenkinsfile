
// Runs Maven inside a container instead of needing it on the agent
def mvn(String args) {
    sh """
        mkdir -p "\$WORKSPACE/.build-home/.m2"
        docker run --rm \\
            -u "\$(id -u):\$(id -g)" \\
            -v "\$PWD":/src \\
            -v "\$WORKSPACE/.build-home":/var/maven \\
            -e HOME=/var/maven \\
            -e MAVEN_CONFIG=/var/maven/.m2 \\
            -w /src \\
            maven:3.9-eclipse-temurin-11 \\
            mvn -B -Duser.home=/var/maven ${args}
    """
}

node {
    // Docker Hub repository, from the job's IMAGE_REPO parameter
    def imageRepo = (params.IMAGE_REPO ?: 'aastardev1/lern-service').trim()

    try {
        stage('Checkout') {
            // Clean everything except the build cache (Maven .m2 + Scala .sbt)
            
            def branch = (params.BRANCH ?: 'spark-v1.0.2.1-work').trim()

            // Uses the repo + branch from the job's SCM config (Branch Specifier: */${BRANCH})
            checkout scm

            def shortCommit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
            // Docker tags can't contain '/', so feature/foo -> feature-foo
            def safeBranch  = branch.replaceAll('[^A-Za-z0-9_.-]', '-')
            env.IMAGE_TAG = "${safeBranch}_${shortCommit}_${env.BUILD_NUMBER}"
            echo "Image tag: ${env.IMAGE_TAG}"
        }

        stage('Maven Build') {
            mvn('clean install -P lern,aws -DskipTests')
        }

        stage('Play2 Dist') {
            mvn('play2:dist -pl modules/lern/service -P lern,aws')
        }

        stage('Docker Build') {
            sh "docker build -f build/lern/Dockerfile --build-arg CSP=aws -t ${imageRepo}:${env.IMAGE_TAG} ."
        }

        stage('Docker Push') {
            sh "docker push ${imageRepo}:${env.IMAGE_TAG}"
        }

        currentBuild.result = 'SUCCESS'
        currentBuild.description = "${imageRepo}:${env.IMAGE_TAG}"

    } catch (err) {
        currentBuild.result = 'FAILURE'
        echo "Build failed: ${err}"
        throw err
    } finally {
        stage('Cleanup') {
            // Remove the local image to save disk space; keep the agent's Docker Hub login
            sh "docker rmi ${imageRepo}:${env.IMAGE_TAG} || true"
        }
    }
}
