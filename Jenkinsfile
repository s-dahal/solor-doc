pipeline {
    agent {
        label 'linux'
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
        stage('Build PDF') {
            steps {
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
        }
    }
}