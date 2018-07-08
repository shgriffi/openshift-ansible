#!/usr/bin/env groovy
/**
  This is the Jenkinsfile for backing up elasticsearch using s3.
  Elasticsearch S3 repository plugin is required.
 */
ansiColor('xterm') {
  timestamps {
        node('dhslave') {
          configFileProvider(
          [configFile( fileId: 'kubeconfig', variable: 'KUBECONFIG')]){
	      stage('Pull Git Repositories') {
                checkout_playbook_repo()
              }
              stage('Trigger Playbook') {
                start_backup()
              }
          }
        }
  }
}
def checkout_playbook_repo() {
    checkout poll: false, scm: [
      $class: 'GitSCM',
      branches: [[name: "*/master"]],
      doGenerateSubmoduleConfigurations: false,
      extensions: [
        [$class: 'WipeWorkspace'],
        [$class: 'RelativeTargetDirectory', relativeTargetDir: 'dh-ci-util']
      ],
      submoduleCfg: [],
      userRemoteConfigs: [[url: 'https://gitlab.cee.redhat.com/data-hub/dh-ci-util.git']]
    ]
}
def start_backup() {
  try {
    // Check if repository exists
    checkRepositoryResponse = sh (
      script: 'curl --insecure --cert $WORKSPACE/dh-ci-util/certificates/stage-es-certs/system.admin.crt --key $WORKSPACE/dh-ci-util/certificates/stage-es-certs/system.admin.key -i -k ${ELASTICSEARCH_ENDPOINT}/_snapshot/${BACKUP_NAME}',
      returnStdout: true
    ).trim()
    
    repoDoesNotExist = checkRepositoryResponse.toLowerCase().contains('404 not found')
    
    // Create the repository if one does not exist
    if (repoDoesNotExist)
    {
        createRepositoryResponse = sh (
            script: 'curl --insecure --cert $WORKSPACE/dh-ci-util/certificates/stage-es-certs/system.admin.crt --key $WORKSPACE/dh-ci-util/certificates/stage-es-certs/system.admin.key -X PUT -H "Content-Type: application/json" -d ' + /'{"type": "s3","settings": {"bucket": "${S3_BUCKET}","base_path": "${S3_PATH}","endpoint": "${S3_ENDPOINT}","access_key":"${S3_ACCESS_KEY}","secret_key":"${S3_SECRET_KEY}"}}'/ + ' -i ${ELASTICSEARCH_ENDPOINT}/_snapshot/${BACKUP_NAME}',
            returnStdout: true
        ).trim()
        
        repositoryCreated = createRepositoryResponse.toLowerCase().contains('200 ok')
        
        if (!repositoryCreated){
            error('Repository could not be created, exiting: ' + createRepositoryResponse)
            return 1
        }
    }
    else
    {
        repoExists = checkRepositoryResponse.toLowerCase().contains('200 ok')
        
        if (!repoExists)
        {
            error('Repository could not be found, exiting: ' + checkRepositoryResponse)
            return 1    
        }
    }
    
    // If the repository exists, take a snapshot
    currDate = new Date().format("yyyyMMdd_hhmm", TimeZone.getTimeZone('UTC'))
    createSnapshotResponse = sh (
        script: 'curl --insecure --cert $WORKSPACE/dh-ci-util/certificates/stage-es-certs/system.admin.crt --key $WORKSPACE/dh-ci-util/certificates/stage-es-certs/system.admin.key -X PUT -H "Content-Type: application/json" -d ' + /'{"indices": "${ELASTICSEARCH_INDICES}"}'/ + ' -i ${ELASTICSEARCH_ENDPOINT}/_snapshot/${BACKUP_NAME}/snapshot_' + currDate,
        returnStdout: true
    ).trim()
        
    snapshotCreated = createSnapshotResponse.toLowerCase().contains('200 ok')
    
    // Return error if snapshot could not be created
    if (!snapshotCreated)
    {
        error('Snapshot could not be created, exiting: ' + createSnapshotResponse)
        return 1
    }
    
    echo "Snapshot created"
  } catch (err) {
    echo 'Exception caught, being re-thrown...'
    throw err
  }
}
