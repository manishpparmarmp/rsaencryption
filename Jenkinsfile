// Common pipeline functions (from Groovy)
def common = null;

String dockerRepo = '319859009076.dkr.ecr.us-east-1.amazonaws.com'
String branchOrTag = null
String apiCloudfrontId = null

pipeline {
  options {
    ansiColor('xterm')
    parallelsAlwaysFailFast()
    withAWS(region: params.AWS_REGION)
  }

  agent {
    node {
      label params.JENKINS_NODE_LABEL
    }
  }

  parameters {
    booleanParam(name: 'REBUILD_PARAMETERS', defaultValue: false, description: 'If selected, the build will automatically exit and build parameters will be rebuilt.')
    choice(name: 'ENVIRONMENT', choices: ['dev01', 'preprd01', 'prd01'], description: 'The environment to provision')
    choice(name: 'ACTION', choices: ['plan', 'apply'], description: 'Which Terraform action to apply <p><pre>plan</pre> displays execution plan - does not build or deploy anything</p><p><pre>apply</pre> performs execution plan</p>')
    string(name: 'VERSION_OVERRIDE', description: 'Which version to deploy (instead of the current branch/tag)')
    booleanParam(name: 'BUILD_API', defaultValue: false, description: 'If selected, the API Docker image will be built, tagged, and stored in ECR. <strong>This will overwrite any existing artifacts!</strong>')
    booleanParam(name: 'BUILD_NGINX', defaultValue: false, description: 'If selected, the NGINX Docker image will be built, tagged, and stored in ECR. <strong>This will overwrite any existing artifacts!</strong>')
    choice(name: 'AWS_REGION', choices: ['us-east-1'], description: 'The default AWS region')
    string(name: 'JENKINS_NODE_LABEL', description: 'The tag of the Jenkins worker node to run the pipeline on', defaultValue: 'arcadiaawsJenkinsKitchenSinkLinux')
    credentials(name: 'GIT_CREDENTIALS', description: 'GitHub credentials to perform the deployment with', defaultValue: 'github-api-arcadia-ci', credentialType: "com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl", required: true)
  }

  stages {
    stage('init') {
      steps {
        script {
          // Load common scripts into global variable
          def subscript = findFiles(glob: '**/jenkins/shared/common.groovy')
          common = load(subscript[0].path)

          // do all the basic initialization steps
          common.initialize(tgenv: true, tfenv: true, nvm: true, git: true, environment: true, env: env, params: params)

          branchOrTag = params.VERSION_OVERRIDE ?: env.TAG_NAME ?: env.BRANCH_NAME
          env.TF_VAR_api_version = branchOrTag
        }
      }
    }

    stage('docker images') {
      when {
        expression { params.ACTION == 'apply' }
      }

      stages {
        stage('nginx') {
          when {
            expression { params.BUILD_NGINX == true }
          }

          steps {
            script {
              def tagName = "${dockerRepo}/patient-chart/nginx:${branchOrTag}"

              common.bash("""
                ${ecrLogin()}
                docker build -f docker/production/nginx/Dockerfile -t ${tagName} docker/production/nginx
                docker push ${tagName}
              """)
            }
          }
        }

        stage('api') {
          when {
            expression { params.BUILD_API == true }
          }

          steps {
            script {
              withCredentials([usernamePassword(credentialsId: params.GIT_CREDENTIALS, passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                def tagName = "${dockerRepo}/patient-chart/api:${branchOrTag}"

                common.bash("""
                  ${ecrLogin()}

                  docker build --build-arg GITHUB_TOKEN=${GIT_PASSWORD} -f docker/production/api/Dockerfile -t ${tagName} .
                  docker push ${tagName}
                """)
              }
            }
          }
        }
      }
    }

    stage('terragrunt action') {
      steps {
        script {
          dir("terragrunt/arcadiaaws/api/${params.ENVIRONMENT}") {
            common.bash("""
              terragrunt init -reconfigure
              terragrunt ${params.ACTION} ${params.ACTION == 'apply' ? '-auto-approve' : ''}
            """)

            if (params.ACTION == 'apply') {
              s3Download(
                bucket: 'arcadia.provisioning',
                path: "apps/patient-chart/arcadiaaws/api/${params.ENVIRONMENT}/terraform.tfstate",
                force: true,
                file: 'api-state.json'
              )

              currentState = readJSON file: 'api-state.json', returnPojo: true
              apiCloudfrontId = currentState?.outputs?.api_cloudfront_id?.value
            }
          }
        }
      }
    }

    stage('invalidate cloudfront') {
      when {
        expression { params.ACTION == 'apply' }
      }

      steps {
        script {
          cfInvalidate(distribution: apiCloudfrontId, paths:['/*'], waitForCompletion: true)
        }
      }
    }
  }

  post {
    cleanup {
      cleanWs disableDeferredWipeout: true, deleteDirs: true
    }
  }
}
