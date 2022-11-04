// Common pipeline functions (from Groovy)
def common = null;

/* Requires the Docker Pipeline plugin */
pipeline {
    agent {
    node {
      label params.JENKINS_NODE_LABEL
    }
  }
   parameters {
    string(name: 'JENKINS_NODE_LABEL', description: 'The tag of the Jenkins worker node to run the pipeline on', defaultValue: 'Linux')
    credentials(name: 'GIT_CREDENTIALS', description: 'GitHub credentials to perform the deployment with', defaultValue: 'rsaencryption', credentialType: "com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl'", required: true)
  }  
    stages {
        stage('build') {
            steps {
                sh 'mvn --version'
            }
        }
    }
}
