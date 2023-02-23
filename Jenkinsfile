def doCPILintStage = false

pipeline {
    agent any

    //Configure the following environment variables before executing the Jenkins Job
    environment {
//        IntegrationFlowID = "Books_Integration_Flow"
        CPIHost = "${env.CPI_HOST}"
        CPIOAuthHost = "${env.CPI_OAUTH_HOST}"
        CPIOAuthCredentials = "${env.CPI_OAUTH_CRED}"
        GITRepositoryURL = "${env.GIT_REPOSITORY_URL}"
        GITCredentials = "${env.GIT_CRED}"
        GITBranch = "${env.GIT_BRANCH_NAME}"
        GITComment = "Integration Artefacts update from CI/CD pipeline"
        GITFolder = "IntegrationContent/IntegrationArtefacts"
    }

    parameters {
        string(name: 'IntegrationFlowID', defaultValue: 'IntegrationFlowID', description: 'Integration Flow ID')
    }

    stages {
        stage('Initialization') {
            steps {
                script {
                    //clean up workspace first
                    echo 'Initialize pipeline'
                    deleteDir()
                }
            }
        }
        stage('Store new versions in Git') {
            steps {
                script {
                    //Synchronize repository to workspace
                    checkout([$class                           : 'GitSCM',
                              branches                         : [[name: env.GITBranch]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions                       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: "."],
                                                                  [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [[$class: 'SparseCheckoutPath', path: env.GITFolder]]]],
                              submoduleCfg                     : [],
                              userRemoteConfigs                : [[credentialsId: env.GITCredentials,
                                                                   url          : 'https://' + env.GITRepositoryURL]]])

                    //get Cloud Integration Oauth token
                    def cpiTokenResponse = httpRequest acceptType: 'APPLICATION_JSON',
                            authentication: env.CPIOAuthCredentials,
                            ignoreSslErrors: false,
                            responseHandle: 'LEAVE_OPEN',
                            timeout: 30,
                            url: 'https://' + env.CPIOAuthHost + '/oauth/token?grant_type=client_credentials'
                    def jsonObj = readJSON text: cpiTokenResponse.content
                    def cpiToken = 'bearer' + ' ' + jsonObj.access_token
                    cpiTokenResponse.close()

                    //get flow version from Cloud Integration tenant
                    def cpiVersionResponse = httpRequest acceptType: 'APPLICATION_JSON',
                            customHeaders: [[maskValue: true, name: 'Authorization', value: cpiToken]],
                            ignoreSslErrors: false,
                            responseHandle: 'LEAVE_OPEN',
                            validResponseCodes: '200:299,404',
                            timeout: 30,
                            url: 'https://' + env.CPIHost + '/api/v1/IntegrationDesigntimeArtifacts(Id=\'' + params.IntegrationFlowID + '\',Version=\'active\')'
                    if (cpiVersionResponse.status == 404) {
                        //Flow not found
                        println("received http 404. Please check the Flow ID you provided in the script. Looks like it does not exist on the tenant.")
                        cpiVersionResponse.close()
                        currentBuild.result = 'FAILURE'
                        return
                    }
                    jsonObj = readJSON text: cpiVersionResponse.content
                    def cpiVersion = jsonObj.d.Version
                    println("Flow version on Cloud Integration tenant: '" + cpiVersion + "'")
                    cpiVersionResponse.close()

                    //get version from source repository
                    def git_version
                    if (fileExists(env.GITFolder + '/' + params.IntegrationFlowID)) {
                        dir(env.GITFolder + '/' + params.IntegrationFlowID + "/META-INF") {
                            def manifestFile = readFile(file: 'MANIFEST.MF')
                            def index = manifestFile.indexOf('Bundle-Version:') + 16
                            def lastindex = manifestFile.indexOf('\n', index)
                            git_version = manifestFile.substring(index, lastindex)
                            println("Flow version in Git: '" + git_version + "'")
                        }
                        //now compare the versions
                        if (git_version.trim().equalsIgnoreCase(cpiVersion.trim())) {
                            println("Versions are same, no action required")
                            //EXIT
                            currentBuild.result = 'SUCCESS'
                            return
                        } else {
                            println("Versions are different")
                        }
                    } else {
                        println("Flow does not exist in Git yet")
                    }

                    //delete the old flow content from the workspace so that only the latest flow version is stored
                    dir(env.GITFolder + '/' + params.IntegrationFlowID) {
                        deleteDir()
                    }

                    //download and extract latest integration flow version from Cloud Integration tenant
                    def tempfile = UUID.randomUUID().toString() + ".zip"
                    println("Download artefact")
                    def cpiFlowResponse = httpRequest acceptType: 'APPLICATION_ZIP',
                            customHeaders: [[maskValue: true, name: 'Authorization', value: cpiToken]],
                            ignoreSslErrors: false,
                            responseHandle: 'LEAVE_OPEN',
                            timeout: 30,
                            outputFile: tempfile,
                            url: 'https://' + env.CPIHost + '/api/v1/IntegrationDesigntimeArtifacts(Id=\'' + params.IntegrationFlowID + '\',Version=\'active\')/$value'
                    def disposition = cpiFlowResponse.headers.toString()
                    def index = disposition.indexOf('filename') + 9
                    def lastindex = disposition.indexOf('.zip', index)
                    def filename = disposition.substring(index + 1, lastindex + 4)
                    def folder = env.GITFolder + '/' + filename.substring(0, filename.indexOf('.zip'))
                    println("Unzip artefact")
                    fileOperations([fileUnZipOperation(filePath: tempfile, targetLocation: folder)])
                    cpiFlowResponse.close()

                    //new version exists - check with cpilint
                    doCPILintStage = true

                    //do not remove the zip
                    //fileOperations([fileDeleteOperation(excludes: '', includes: tempfile)])

                    //adding the content to the index and uploading it to the repository
                    dir(folder) {
                        sh 'git add .'
                    }
                    println("Store artefact in Git")
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: env.GITCredentials, usernameVariable: 'GitUserName', passwordVariable: 'GitPassword']]) {
                        sh 'git diff-index --quiet HEAD || git commit -am ' + '\'' + env.GITComment + '\''
                        sh('git push https://${GitUserName}:${GitPassword}@' + env.GITRepositoryURL + ' HEAD:' + env.GITBranch)
                    }
                }
            }
        }

        stage('Check with CPILint') {
            when {
                expression {
                    doCPILintStage
                }
            }
            steps {
                script {
                    //checkout CPILint and rules
                    checkout([$class                           : 'GitSCM',
                              branches                         : [[name: env.GITBranch]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions                       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: '.'],
                                                                  [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [[$class: 'SparseCheckoutPath', path: 'cpilint']]]],
                              submoduleCfg                     : [],
                              userRemoteConfigs                : [[credentialsId: env.GITCredentials,
                                                                   url          : 'https://' + env.GITRepositoryURL]]])

                    //check with cpilint
                    dir('.') {
                        catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                            bat './cpilint/bin/cpilint -rules ./cpilint/rules/rules.xml -directory ./'
                        }
                    }

                    //remove the zip
                    fileOperations([folderDeleteOperation(folderPath: 'cpilint')])
                }
            }
        }
        stage('Cleanup') {
            steps {
                script {
                    //clean up workspace
                    echo 'Clean pipeline'
                    deleteDir()
                }
            }
        }
    }
}