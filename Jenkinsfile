def repoName = "ws-aws-ex1"                         //Repo to store TF code for the TFE Workspace
def repoSshUrl = "git@github.com:TedSpinks/ws-aws-ex1.git"   //Must be ssh since we're using sshagent()
def tfCodeId = "example-${env.BUILD_NUMBER}"        //Unique name to use in the TF code filename and resource
def tfCodeFilePath = "${repoName}/${tfCodeId}.tf"   //Path and filename of the new TF code file
//Credentials
def gitCredentials = 'github-jenkins-aws-example'   //Credential ID in Jenkins of your GitHub Credentials
def tfeCredentials = 'ptfe'                         //Credential ID in Jenkins of your Terraform Enterprise Credentials

pipeline {
  agent {
    label 'general'
  }

  stages {

    stage('1. Add new terraform code to Workspace'){
      steps {
        echo "Pull down any existing Terraform code for this pipeline's Workspace..."
        sshagent (credentials: [gitCredentials]) {
          sh """
              #Clear old Workspace directory - VERY important to make sure no old Terraform code or config files are hanging around!
              rm -rf ${repoName} || true
              #Pull down current Workspace code
              git clone ${repoSshUrl}  #must be an ssh path, since we're using sshagent()
          """
        }
        echo "Add a new Terraform file for this particular pipeline build..."
        script {
          tfCode = """
            resource "aws_instance" "${tfCodeId}" { 
              ami           = "ami-40d28157"  #Ubuntu Server 16.04 LTS HVM SSD
              instance_type = "t2.micro" 
            } 
          """
        }
        writeFile file: tfCodeFilePath, text: tfCode
        sshagent (credentials: [gitCredentials]) {
          sh """
              cd ${repoName}
              git add *.tf
              git commit -m 'New EC2 instance(s) for Jenkins job "${env.JOB_NAME}", build ${env.BUILD_NUMBER}'
              git push origin master
          """
        }
      }
    }

    stage('2. Run the Workspace'){
      environment {
          REPO_NAME = "${repoName}"
      }
      steps {
        /* 
        Note: I originally had the TFE token stored in Jenkins as a "Secret text" credential, but that seemed to corrupt 
        the contents of the string (auth failed), so I recreated it as a "Username with password" credential. Original line:
        withCredentials([string(credentialsId: tfeCredentials, variable: 'TOKEN')]) {
        */
        withCredentials([usernamePassword(credentialsId: tfeCredentials, usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
          sh '''
            cd $REPO_NAME
            terraform init -backend-config="token=$TOKEN"  #Uses config.tf and the user API token to connect to TFE
            terraform apply
          '''
        }
      }
    }

  } //stages

  post {
    always {
      archiveArtifacts artifacts: "${repoName}/*.tf", fingerprint: true
    }
  }

}