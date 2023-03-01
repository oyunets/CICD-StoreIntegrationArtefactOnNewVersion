def isNewVersion = false
def cpiVersion
def iFlowFile

pipeline {
    agent any

    //Configure the following environment variables before executing the Jenkins Job
    parameters {
        string(name: 'IntegrationPackageID', defaultValue: 'IntegrationPackageID', description: 'Integration Package ID')
        string(name: 'IntegrationFlowID', defaultValue: 'IntegrationFlowID', description: 'Integration Flow ID')
        string(name: 'CPIHost', defaultValue: 'CPIHost', description: 'CPIHost')
        string(name: 'CPIOAuthHost', defaultValue: 'CPIOAuthHost', description: 'CPIOAuthHost')
        string(name: 'CPIOAuthCredentials', defaultValue: 'CPIOAuthCredentials', description: 'CPIOAuthCredentials')

        string(name: 'GITRepositoryURL', defaultValue: 'GITRepositoryURL', description: 'GITRepositoryURL')
        string(name: 'GITCredentials', defaultValue: 'GITCredentials', description: 'GITCredentials')
        string(name: 'GITBranch', defaultValue: 'GITBranch', description: 'GITBranch')
        string(name: 'GITComment', defaultValue: 'Integration Artefacts update from CI/CD pipeline', description: 'GITComment')
        string(name: 'GITFolder', defaultValue: 'IntegrationContent/IntegrationArtefacts', description: 'GITFolder')

        string(name: 'CreatetedBy', defaultValue: 'CreatetedBy', description: 'CreatetedBy')
        string(name: 'ModifiedBy', defaultValue: 'ModifiedBy', description: 'ModifiedBy')
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

        stage('Check for new version') {
            steps {
                script {
                    //Synchronize repository to workspace
                    checkout([$class                           : 'GitSCM',
                              branches                         : [[name: params.GITBranch]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions                       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: "."],
                                                                  [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [[$class: 'SparseCheckoutPath', path: params.GITFolder]]]],
                              submoduleCfg                     : [],
                              userRemoteConfigs                : [[credentialsId: params.GITCredentials,
                                                                   url          : 'https://' + params.GITRepositoryURL]]])

                    //get Cloud Integration Oauth token
                    def cpiTokenResponse = httpRequest acceptType: 'APPLICATION_JSON',
                            authentication: params.CPIOAuthCredentials,
                            ignoreSslErrors: false,
                            responseHandle: 'LEAVE_OPEN',
                            timeout: 30,
                            url: 'https://' + params.CPIOAuthHost + '/oauth/token?grant_type=client_credentials'
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
                            url: 'https://' + params.CPIHost + '/api/v1/IntegrationDesigntimeArtifacts(Id=\'' + params.IntegrationFlowID + '\',Version=\'active\')'
                    if (cpiVersionResponse.status == 404) {
                        //Flow not found
                        println("received http 404. Please check the Flow ID you provided in the script. Looks like it does not exist on the tenant.")
                        cpiVersionResponse.close()
                        currentBuild.result = 'FAILURE'
                        return
                    }
                    jsonObj = readJSON text: cpiVersionResponse.content
                    cpiVersion = jsonObj.d.Version
                    println("Flow version on Cloud Integration tenant: '" + cpiVersion + "'")
                    cpiVersionResponse.close()

                    //get version from source repository
                    def git_version
                    if (fileExists(params.GITFolder + '/' + params.IntegrationFlowID)) {
                        dir(params.GITFolder + '/' + params.IntegrationFlowID + "/META-INF") {
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
                    dir(params.GITFolder + '/' + params.IntegrationFlowID) {
                        deleteDir()
                    }

                    //new version exists
                    isNewVersion = true
                }
            }
        }

        stage('Store new versions in Git') {
//            when {
//                expression {
//                    isNewVersion
//                }
//            }
            steps {
                script {
                    //get Cloud Integration Oauth token
                    def cpiTokenResponse = httpRequest acceptType: 'APPLICATION_JSON',
                            authentication: params.CPIOAuthCredentials,
                            ignoreSslErrors: false,
                            responseHandle: 'LEAVE_OPEN',
                            timeout: 30,
                            url: 'https://' + params.CPIOAuthHost + '/oauth/token?grant_type=client_credentials'
                    def jsonObj = readJSON text: cpiTokenResponse.content
                    def cpiToken = 'bearer' + ' ' + jsonObj.access_token
                    cpiTokenResponse.close()

                    //download and extract latest integration flow version from Cloud Integration tenant
                    iFlowFile = UUID.randomUUID().toString() + ".zip"

                    println("Download artefact")
                    def cpiFlowResponse = httpRequest acceptType: 'APPLICATION_ZIP',
                            customHeaders: [[maskValue: true, name: 'Authorization', value: cpiToken]],
                            ignoreSslErrors: false,
                            responseHandle: 'LEAVE_OPEN',
                            timeout: 30,
                            outputFile: iFlowFile,
                            url: 'https://' + params.CPIHost + '/api/v1/IntegrationDesigntimeArtifacts(Id=\'' + params.IntegrationFlowID + '\',Version=\'' + cpiVersion + '\')/$value'
                    def disposition = cpiFlowResponse.headers.toString()
                    def index = disposition.indexOf('filename') + 9
                    def lastindex = disposition.indexOf('.zip', index)
                    def filename = disposition.substring(index + 1, lastindex + 4)
                    def folder = params.GITFolder + '/' + filename.substring(0, filename.indexOf('.zip'))
                    println("Unzip artefact")
                    fileOperations([fileUnZipOperation(filePath: iFlowFile, targetLocation: folder)])
                    cpiFlowResponse.close()

                    //adding the content to the index and uploading it to the repository
                    dir(folder) {
                        sh 'git add .'
                    }
                    println("Store artefact in Git")
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: params.GITCredentials, usernameVariable: 'GitUserName', passwordVariable: 'GitPassword']]) {
                        sh 'git diff-index --quiet HEAD || git commit -am ' + '\'' + params.GITComment + '\''
                        sh('git push https://${GitUserName}:${GitPassword}@' + params.GITRepositoryURL + ' HEAD:' + params.GITBranch)
                    }
                }
            }
        }

        stage('Check with CPILint') {
//            when {
//                expression {
//                    isNewVersion
//                }
//            }
            steps {
                script {
                    println("Get CPILint from SCM")
                    checkout([$class                           : 'GitSCM',
                              branches                         : [[name: params.GITBranch]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions                       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: '.'],
                                                                  [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [[$class: 'SparseCheckoutPath', path: 'cpilint']]]],
                              submoduleCfg                     : [],
                              userRemoteConfigs                : [[credentialsId: params.GITCredentials,
                                                                   url          : 'https://' + params.GITRepositoryURL]]])

                    println("Check iFlow")
                    dir('.') {
                        try {
                            def result = bat(script: '@./cpilint/bin/cpilint -rules ./cpilint/rules/rules.xml -directory ./', label: "Check for optional rules", returnStatus: true, returnStdout: true)
                            echo result;
                        }
                        catch (err) {
                            echo err.getMessage()
                            currentBuild.result = 'UNSTABLE'
                        }
                    }

                    println("Cleanup")
                    fileOperations([folderDeleteOperation(folderPath: 'cpilint')])
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    isNewVersion
                }
            }
            steps {
                script {
                    //get token
                    println("Requesting token from Cloud Integration tenant");
                    def getTokenResp = httpRequest acceptType: 'APPLICATION_JSON',
                            authentication: params.CPIOAuthCredentials,
                            contentType: 'APPLICATION_JSON',
                            httpMode: 'POST',
                            responseHandle: 'LEAVE_OPEN',
                            timeout: 30,
                            url: 'https://' + params.CPIOAuthHost + '/oauth/token?grant_type=client_credentials';
                    def jsonObjToken = readJSON text: getTokenResp.content
                    def token = "Bearer " + jsonObjToken.access_token

                    //deploy integration flow
                    println("Deploying integration flow");
                    def deployResp = httpRequest httpMode: 'POST',
                            customHeaders: [
                                    [maskValue: false, name: 'Authorization', value: token]
                            ],
                            ignoreSslErrors: true,
                            timeout: 30,
                            url: 'https://' + params.CPIHost + '/api/v1/DeployIntegrationDesigntimeArtifact?Id=\'' + params.IntegrationFlowID + '\'&Version=\'' + cpiVersion + '\'';

                    Integer counter = 0;
                    def deploymentStatus;
                    def continueLoop = true;
                    println("Deployment successful triggered. Checking status.");
                    //performing the loop until we get a final deployment status.
                    while (counter < 20 & continueLoop == true) {
                        timeout(time: 3, unit: 'SECONDS') {
                            counter = counter + 1;
                            def statusResp = httpRequest acceptType: 'APPLICATION_JSON',
                                    customHeaders: [
                                            [maskValue: false, name: 'Authorization', value: token]
                                    ],
                                    httpMode: 'GET',
                                    responseHandle: 'LEAVE_OPEN',
                                    timeout: 30,
                                    url: 'https://' + params.CPIHost + '/api/v1/IntegrationRuntimeArtifacts(\'' + params.IntegrationFlowID + '\')';
                            def jsonObj = readJSON text: statusResp.content;
                            deploymentStatus = jsonObj.d.Status;

                            println("Deployment status: " + deploymentStatus);
                            if (deploymentStatus.equalsIgnoreCase("Error")) {
                                //get error details
                                def deploymentErrorResp = httpRequest acceptType: 'APPLICATION_JSON',
                                        customHeaders: [
                                                [maskValue: false, name: 'Authorization', value: token]
                                        ],
                                        httpMode: 'GET',
                                        responseHandle: 'LEAVE_OPEN',
                                        timeout: 30,
                                        url: 'https://' + params.CPIHost + '/api/v1/IntegrationRuntimeArtifacts(\'' + params.IntegrationFlowID + '\')' + '/ErrorInformation/$value';
                                def jsonErrObj = readJSON text: deploymentErrorResp.content
                                def deployErrorInfo = jsonErrObj.parameter;
                                println("Error Details: " + deployErrorInfo);
                                statusResp.close();
                                deploymentErrorResp.close();
                                error("Integration flow not deployed successfully. Ending job now.");
                            } else if (deploymentStatus.equalsIgnoreCase("Started")) {
                                println("Integration flow deployment was successful")
                                statusResp.close();
                                continueLoop = false
                            } else {
                                println("The integration flow is not yet started. Will wait 3s and then check again.")
                            }
                        }
                    }
                    if (!deploymentStatus.equalsIgnoreCase("Started")) {
                        error("No final deployment status could be reached. Kindly check the tenant for any issue.");
                    }
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