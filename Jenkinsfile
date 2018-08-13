/*******************************************************************************
 * Copyright (c) 2018 Red Hat Inc
 *
 * This program and the accompanying materials are made available under the
 * terms of the Eclipse Public License 2.0 which is available at
 * http://www.eclipse.org/legal/epl-2.0
 *
 * SPDX-License-Identifier: EPL-2.0
 *******************************************************************************/
 
import groovy.transform.Field

/*
 * The ID of the Jenkins credentials object (type SSH) which holds the key for
 * the machine created by OpenStack. You may have need to update this and/or
 * create your own credentials object in Jenkins.
 */
def masterCredentialsId = 'c0eb724c-f8ab-418f-b4b6-336e779beeac'
def openshiftCredentialsId = 'd547b54a-3fae-4a00-90b4-dab724eb8742'
def gitUrl
def gitBranch
def gitCommit

@Field masterMachine

def master ( params ) {
    def host = masterMachine.address
    def sshOpts = "-oConnectTimeout=15 -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -i '${params.identity}' '${params.user}@${host}'"
    def escaped = escape(params.script)
    return sh ( returnStatus: params.returnStatus ?: false , script: "ssh $sshOpts $escaped" )
}

def escape ( String input ) {
    if ( input == null ) {
        return null
    }

    return"'" + input.replace("'", "'\"'\"'") + "'"
}

properties([
    buildDiscarder(
        logRotator(artifactNumToKeepStr: '10')
    ),
    disableConcurrentBuilds(),
    disableResume(),
    [$class: 'GithubProjectProperty', projectUrlStr: 'https://github.com/redhat-iot/hono-ci/']
])

node {

    stage ('Checkout') {
        def scmInfo = checkout scm
        echo "${scmInfo}"
        gitUrl = scmInfo.GIT_URL
        gitBranch = scmInfo.GIT_BRANCH.replaceAll('^origin\\/', '')
        gitCommit = scmInfo.GIT_COMMIT
        echo "hono-ci - url: ${gitUrl}, branch: ${gitBranch}, commit: ${gitCommit}"
    }

    stage ('Build') {
        git 'https://github.com/eclipse/hono.git'
        withMaven(
            maven: 'apache-maven-3.5.x',
            mavenLocalRepo: '.repository'
        ) {
            dir('.repository') {
                deleteDir()
            }
            sh 'mvn install -B -DskipTests -DcreateJavadoc=true'
        }
    }

    parallel (
    
        'Integration Tests': {
    
            stage ('Verify') {
                node('container-build') {
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
        },

        'Deploy Tests': {

            stage('Create Machine') {
                masterMachine = openstackMachine cloud: 'Upshift CI', template: 'openshift-deploy-test'
                echo "Machine: ${masterMachine.address}"
            }

            node('openshift-tester') {
                stage('Wait for Startup') {
                    timeout(5) {
                        waitUntil {
                            def rc = sh ( returnStatus: true, script: "ping -qc 5 ${masterMachine.address}" )
                            echo "Ping result: $rc"
                            return rc == 0
                        }
                    }
                    timeout(30) {
                        withCredentials([sshUserPrivateKey(credentialsId: masterCredentialsId, keyFileVariable: 'identity', usernameVariable: 'user')]) {
                            waitUntil {
                                def rc = master ( identity: identity, user: user, returnStatus: true, script: "systemctl status -q docker" )
                                echo "SSH result: $rc"
                                return rc == 0
                            }
                        }
                    }
                    withCredentials([sshUserPrivateKey(credentialsId: masterCredentialsId, keyFileVariable: 'identity', usernameVariable: 'user')]) {
                        master ( identity: identity, user: user, returnStatus: true, script: "ip a" )
                    }
                }

                stage('Deploy OKD') {
                    withCredentials([
                          sshUserPrivateKey(credentialsId: masterCredentialsId, keyFileVariable: 'identity', usernameVariable: 'user'),
                          usernamePassword(credentialsId: openshiftCredentialsId, usernameVariable: 'openshiftUser', passwordVariable: 'openshiftPassword')
                        ]) {
                        master ( identity: identity, user: user, script: "git clone -b ${escape(gitBranch)} ${escape(gitUrl)} hono-ci" )
                        master ( identity: identity, user: user, script: "cd hono-ci && git checkout ${escape(gitCommit)} && git submodule update --init --recursive" )
                        master ( identity: identity, user: user, script: "cd hono-ci/tests/test-deploy && ./run ${masterMachine.address} ${escape(openshiftUser)} ${escape(openshiftPassword)}" )
                    }
                }

                stage('Setup OC client tools') {
                    sh("curl -Ls https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz -o /root/openshift-origin-client-tools.tar.gz")
                    sh("cd /root && tar xzf /root/openshift-origin-client-tools.tar.gz")
                    sh("cp /root/openshift-origin-client-tools-*-linux-64bit/oc /usr/local/bin")
                    sh("oc version")
                }

                // now OKD and the tester node are running

                stage('Test deployment') {

                    def cdir = pwd()

                    withEnv(["KUBECONFIG=${cdir}/kubeconfig"]) {

                        // Login to OKD cluster
                        sh('echo $KUBECONFIG')
                        sh("oc login --username developer --password developer --insecure-skip-tls-verify=true https://${masterMachine.address}:8443")
                        sh("oc login --username admin --password admin12 --insecure-skip-tls-verify=true https://${masterMachine.address}:8443")
                        sh("oc login --username developer")
                        sh('cat $KUBECONFIG')
                        sh('df .')

                        // get Hono to local tester node
                        dir('hono') {
                            git 'https://github.com/eclipse/hono.git'

                            // change to deploy directory, as described in the documentation
                            dir('deploy/src/main/deploy/openshift_s2i') {

                                stage('Deploy EnMasse') {

                                    sh('echo $KUBECONFIG')
                                    sh('cat $KUBECONFIG')
                                    // create new enmasse project
                                    sh("oc login -u admin")
                                    sh("oc new-project enmasse-infra --display-name='EnMasse Instance'")

                                    // download and unpack EnMasse
                                    sh("curl -LO https://github.com/EnMasseProject/enmasse/releases/download/0.24.0/enmasse-0.24.0.tgz")
                                    sh("tar xzf enmasse-0.24.0.tgz")

                                    // Deploy EnMassee
                                    sh("oc apply -f enmasse-0.24.0/install/bundles/enmasse-with-standard-authservice")

                                }

                                stage('EnMasse ready') {

                                    timeout(10) {
                                        waitUntil {
                                            def num1 = sh(returnStdout: true, script: 'oc -n enmasse-infra get deploy/api-server --template={{.status.readyReplicas}} || true')
                                            def num2 = sh(returnStdout: true, script: 'oc -n enmasse-infra get deploy/keycloak --template={{.status.readyReplicas}} || true')
                                            return num1+num2 == "11"
                                        }
                                    }
                                    sh("oc login -u developer")
                                }

                                stage('Deploy Hono') {

                                    sh('oc new-project hono --display-name="Eclipse Hono"')
                                    sh('oc create configmap influxdb-config --from-file="../influxdb.conf"')
                                    sh('oc label configmap/influxdb-config app=hono-metrics')
                                    sh('oc process -f hono-template.yml -p ENMASSE_NAMESPACE=enmasse | oc create -f -')

                                }

                                stage('Deploy Grafana') {

                                    sh('oc new-project grafana --display-name="Grafana Dashboard"')
                                    sh('oc create configmap grafana-provisioning-dashboards --from-file=../../config/grafana/provisioning/dashboards')
                                    sh('oc create configmap grafana-dashboard-defs --from-file=../../config/grafana/dashboard-definitions')
                                    sh('oc label configmap grafana-provisioning-dashboards app=hono-metrics')
                                    sh('oc label configmap grafana-dashboard-defs app=hono-metrics')
                                    sh('oc process -f grafana-template.yml -p ADMIN_PASSWORD=admin | oc create -f -')

                                }

                                stage('Wait for Hono') {
                                    timeout(10) {t
                                        waitUntil {
                                            def num1 = sh(returnStdout: true, script: 'oc -n hono get dc/hono-adapter-http-vertx --template={{.status.readyReplicas}} || true')
                                            def num2 = sh(returnStdout: true, script: 'oc -n hono get dc/hono-adapter-mqtt-vertx --template={{.status.readyReplicas}} || true')
                                            echo "http: $num1, mqtt: $num2"
                                            return num1 == "1" && num2 == "1"
                                        }
                                    }
                                    sh("oc -n hono get all || true")
                                    sh("oc -n hono describe dc/hono-adapter-http-vertx || true")
                                    timeout(60) {
                                        waitUntil {
                                            def num1 = sh(returnStdout: true, script: 'oc -n hono get dc/hono-adapter-http-vertx --template={{.status.readyReplicas}} || true')
                                            def num2 = sh(returnStdout: true, script: 'oc -n hono get dc/hono-adapter-mqtt-vertx --template={{.status.readyReplicas}} || true')
                                            echo "http: $num1, mqtt: $num2"
                                            return num1 == "1" && num2 == "1"
                                        }
                                    }
                                }

                            } // end-dir openshift_s2i

                        } // end-dir hono
                    }
                } // end-stage Test Deployment

                stage ('Run JMeter') {

                    sh('curl -LO http://www-us.apache.org/dist/jmeter/binaries/apache-jmeter-4.0.tgz')
                    sh('tar xavf apache-jmeter-4.0.tgz')

                    dir('apache-jmeter-4.0') {
                        sh('cp -v ../jmeter/target/plugin/*.jar lib/ext/')
                        sh('mkdir results')

                        def clusterDomain = "${masterMachine.address}.nip.io"

                        def cmd = ""

                        cmd+="bin/jmeter -n"

                        cmd+=" -t ../jmeter/src/jmeter/http_messaging_throughput_test.jmx"
                        cmd+=" -t ../results/http_messaging_throughput_test.jtl"

                        cmd+=" -Jjmeterengine.stopfail.system.exit=true -JdeviceCount=10 -JconsumerCount=2"

                        cmd+=" -Lorg.eclipse.hono.client.impl=WARN -Lorg.eclipse.hono.jmeter=INFO"
                        cmd+=" -Jrouter.host=messaging-enmasse.${clusterDomain} -Jrouter.port=443"
                        cmd+=" -Jregistration.host=hono-service-device-registry-http-hono.${clusterDomain} -Jregistration.http.port=80"
                        cmd+=" -Jmqtt.host=hono-adapter-mqtt-vertx-sec-hono.${clusterDomain} -Jmqtt.port=443"
                        cmd+=" -Jhttp.host=hono-adapter-http-vertx-sec-hono.${clusterDomain} -Jhttp.port=443"

                        echo(cmd)
                        // sh(cmd)

                        perfReport compareBuildPrevious: true, failBuildIfNoResultFile: false, modeThroughput: true, percentiles: '0,50,90,100', sourceDataFiles: 'results/*.jtl'
                    }

                }

            } // end-node openshift-tester

        } // end-deployTest

    ) // end-parallel

}
