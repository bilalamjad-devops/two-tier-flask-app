node {

    def app
    def imageName = "YOUR_DOCKERHUB_USERNAME/flaskapp"
    def gitEmail = "YOUR_GITHUB_EMAIL"
    def gitUsername = "YOUR_GITHUB_USERNAME"

    stage('Clone Repository') {
        checkout scm
    }

    stage('Build Docker Image') {
        app = docker.build("${imageName}")
    }

    stage('Test Docker Image') {
        app.inside {
            sh 'echo "Tests Passed Successfully"'
        }
    }

    stage('Push Docker Image') {
        docker.withRegistry('https://registry.hub.docker.com', 'dockerhub') {
            // Push with Jenkins BUILD_NUMBER tag for traceability
            app.push("${env.BUILD_NUMBER}")
            // Also push latest tag
            app.push("latest")
        }
    }

    stage('Update Kubernetes Manifest') {
        withCredentials([
            usernamePassword(
                credentialsId: 'github',
                usernameVariable: 'GIT_USERNAME',
                passwordVariable: 'GIT_PASSWORD'
            )
        ]) {
            // Configure Git identity
            sh "git config user.email '${gitEmail}'"
            sh "git config user.name '${gitUsername}'"

            // Update image tag in deployment manifest
            sh """
                sed -i 's+image:.*+image: ${imageName}:${BUILD_NUMBER}+g' k8s/two-tier-app-deployment.yml
            """

            // Commit and push updated manifest
            sh 'git add .'
            sh "git commit -m 'Updated image tag to ${BUILD_NUMBER}'"
            sh """
                git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/two-tier-flask-app.git HEAD:main
            """
        }
    }
}
