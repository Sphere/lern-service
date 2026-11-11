// Jenkins Scripted Pipeline for lern-service (AWS)
// Image tag = <branch>_<short commit>_<build number>
//   e.g. spark-v1.0.2.1-work_0477f61_42
//
// Prerequisite: the Jenkins agent must already be logged in to Docker Hub
// as the jenkins user (run once on the agent: sudo -u jenkins docker login -u aastardev1)

properties([
    parameters([
        string(name: 'BRANCH', defaultValue: 'spark-v1.0.2.1-work',
               description: 'Branch of Sphere/lern-service to build'),
        string(name: 'COMMIT', defaultValue: '0477f61',
               description: 'Commit to build. Leave empty to build the branch tip (adds 8288d34, SMS OTP fix)')
    ])
])

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
    def repoUrl   = 'https://github.com/Sphere/lern-service.git'
    def imageRepo = 'aastardev1/lern-service'
    def imageTag  = ''

    try {
        stage('Checkout') {
        

            def branch = (params.BRANCH ?: 'spark-v1.0.2.1-work').trim()
            def commit = (params.COMMIT ?: '').trim()

            git branch: branch, url: repoUrl

            if (commit) {
                echo "Pinning build to commit ${commit}"
                sh "git checkout ${commit}"
            } else {
                echo "Building tip of ${branch}"
            }

            def shortCommit = sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
            // Docker tags can't contain '/', so feature/foo -> feature-foo
            def safeBranch  = branch.replaceAll('[^A-Za-z0-9_.-]', '-')
            imageTag = "${safeBranch}_${shortCommit}_${env.BUILD_NUMBER}"
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
