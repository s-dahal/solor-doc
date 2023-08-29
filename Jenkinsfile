@Library("titan-library") _

pipeline {
    agent {
        label 'linux'
    }
    triggers {
        //cron(cron_string)
        gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All')
    }
    options {
        // Set this to true if you want to clean workspace during the prep stage
        skipDefaultCheckout(true)

        // Console debug options
        timestamps()
        ansiColor('xterm')

        gitLabConnection('gitlab-installer-connection')
    }
    tools {
        jdk 'default'
        maven 'default'
    }
    stages {
        stage('Initialize') {
            steps {
                // Clean before checkout / build
                cleanWs()
                checkout scm

                // Get the release information
                script {
                    pomModel = readMavenPom(file: 'pom.xml')
                    pomVersion = pomModel.getVersion()
                    isSnapshot = pomVersion.contains("-SNAPSHOT")
                    pomGroupId = pomModel.groupId
                    pomArtifactId = pomModel.artifactId

                    echo "pomVersion: ${pomVersion}"
                    echo "isSnapshot: ${isSnapshot}"
                    echo "pomGroupId: ${pomGroupId}"
                    echo "pomArtifactId: ${pomArtifactId}"
                }
            }
        }
        stage('Build Prerequisite Docs') {
            steps {
                updateGitlabCommitStatus name: 'build', state: 'running'
                sh """
                    mvn -f ANF clean install
                    mvn -f tinkar-doc clean install
                """
            }
        }
        stage('Build PDF') {
            steps {
                updateGitlabCommitStatus name: 'build', state: 'running'
                sh """
                    mvn clean install
                """
            }
        }
        stage('Deploy PDF for Release Versions') {
            when{
                expression{
                    !isSnapshot && BRANCH_NAME == 'main'
                }
            }
            steps {
                updateGitlabCommitStatus name: 'deploy', state: 'running'
                configFileProvider([configFile(fileId: 'settings.xml', variable: 'MAVEN_SETTINGS')]) {
                    sh """
                        mvn deploy:deploy-file \
                            --batch-mode \
                            -e \
                            -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn \
                            -s ${MAVEN_SETTINGS} \
                            -Dfile=\$(find statements/target/site/docbook -name symbolic-information-analytics*.pdf) \
                            -Durl=https://nexus.build.tinkarbuild.com/repository/maven-releases/ \
                            -DgroupId=${pomGroupId} \
                            -DartifactId=${pomArtifactId} \
                            -Dversion=${pomVersion} \
                            -Dtype=pdf \
                            -Dpackaging=pdf \
                            -DrepositoryId=titan-maven-releases
                    """
                }
            }
            post {
                failure {
                    updateGitlabCommitStatus name: 'deploy', state: 'failed'
                }
                success {
                    updateGitlabCommitStatus name: 'deploy', state: 'success'
                }
                aborted {
                    updateGitlabCommitStatus name: 'deploy', state: 'canceled'
                }
            }
        }
    }
    post {
        failure {
            updateGitlabCommitStatus name: 'build', state: 'failed'
        }
        success {
            updateGitlabCommitStatus name: 'build', state: 'success'
        }
        aborted {
            updateGitlabCommitStatus name: 'build', state: 'canceled'
        }
    }
}
