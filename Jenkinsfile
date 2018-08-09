pipeline {

    agent any

    tools {
        maven 'apache-maven-3.5.x'
    }

    stages {

        stage ('Build') {
            steps {
                git 'https://github.com/eclipse/hono.git'
                withMaven(
                    maven: 'apache-maven-3.5.x',
                    mavenLocalRepo: '.repository'
                ) {
                    sh 'mvn install -B -DskipTests -DcreateJavadoc=true'
                }
            }
        }

        stage ('Verify'){
            agent {
                label 'container-build'
            }
            steps {
                git 'https://github.com/eclipse/hono.git'
                withMaven(
                    maven: 'apache-maven-3.5.x',
                    mavenLocalRepo: '.repository'
                ) {
                    sh 'mvn install verify -B -DcreateJavadoc=true'
                }
                junit '**/target/surefire-reports/**/*.xml'
                jacoco()
            }
        }
    }
}