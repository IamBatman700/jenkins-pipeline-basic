pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Build stage'
                sh 'pwd'
                sh 'ls -la'
                sh 'touch build.txt'
            }
        }

        stage('Test') {
            steps {
                echo 'Test stage'
                sh 'ls -la'
                sh 'touch test.txt'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploy stage'
                sh 'mkdir -p deploy'
                sh 'mv build.txt deploy/'
                sh 'mv test.txt deploy/'
                sh 'ls -la deploy'
            }
        }
    }
}
