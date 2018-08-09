pipeline {

    agent any

    tools {
        maven 'apache-maven-3.5.x'
    }

    stages {

        stage ('Checkout') {
            steps {
                git 'https://github.com/eclipse/hono.git'
            }
        }

        stage ('Prepare') {
            steps {
                echo 'Nothing to prepare right now'
            }
        }

        stage ('Build') {
            steps {
                withMaven(
                    maven: 'apache-maven-3.5.x',
                    mavenLocalRepo: '.repository'
                ) {
                    sh 'mvn install -B -DcreateJavadoc=true'
                }
            }
        }

        stage ('Collect') {
            steps {
                junit '**/target/surefire-reports/**/*.xml'
                jacoco()
            }
        }
    }
}