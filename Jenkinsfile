// Job 1: CI Pipeline (Build & Push)

node {
    def app

    stage('Clone repository') {
        // Pulls code from the Git repo configured in the Jenkins job UI
        checkout scm
    }

    stage('Build image') {
       // 👉 Replace 'Your-Docker-Hub-Username/test' with your actual Docker Hub "username/repo"
       app = docker.build("bilalamjaddevops/test")
    }

    stage('Test image') {
        app.inside {
            // Add your actual test commands here (e.g., sh 'npm test' or sh 'pytest')
            sh 'echo "Tests passed"'
        }
    }

    stage('Push image') {
        // 'dockerhub' must match the ID of your credentials stored in Jenkins
        docker.withRegistry('https://registry.hub.docker.com', 'dockerhub') {
            // Uses Jenkins Build Number as the tag (e.g., version 1, 2, 3...)
            app.push("${env.BUILD_NUMBER}")
        }
    }
    

}
