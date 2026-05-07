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


    stage('Update Manifest') {
            script {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    // 'github' must match your Jenkins Credentials ID for Git
                    withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        
                        //👉 CHANGE: Set your own email and name for Git commits - Your-GitHub-email@example.com, Your-GitHub-Name
                        sh "git config user.email devopsengineerbilalamjad@gmail.com"
                        sh "git config Bilal Amjad "
                        
                        // This updates the 'image' field in deployment.yaml to the new tag
                        //👉 CHANGE: Ensure the regex 'Your-Docker-Hub-Username/test.*' matches your YAML image string
                        sh "sed -i 's+bilalamjaddevops/test.*+bilalamjaddevops/test:${DOCKERTAG}+g' deployment.yaml"
                        
                        sh "git add ."
                        sh "git commit -m 'Done by Jenkins Job changemanifest: ${env.BUILD_NUMBER}'"
                        
                        // Ensure 'kubernetesmanifest.git' and the branch 'main' match your repo
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/kubernetesmanifest.git HEAD:main"
      }
    }

}
