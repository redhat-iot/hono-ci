node {

    def mavenLocalRepo = "~/.hono.m2repo"

    stage ('Checkout') {
        git 'https://github.com/eclipse/hono.git'
    }
    
    stage ('Prepare') {
        dir(mavenLocalRepo) {
            deleteDir()
        }
    }

    stage ('Build') {
        withMaven(
            maven: 'Maven 3.5.x',
            mavenLocalRepo: mavenLocalRepo
        ) {
            sh 'mvn install -B -Dtest.env=true -DcreateJavadoc=true'
        }
    }

    stage ('Post') {
        junit '**/target/surefire-reports/**/*.xml'
        jacoco()
    }
}