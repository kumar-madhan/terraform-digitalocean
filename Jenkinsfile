pipeline {
    agent {
        label 'Build_Agent'
    }

    parameters {
        choice(name: 'ACTION', choices: ['apply', 'destroy'], description: 'Choose whether to apply or destroy infrastructure')
        string(name: 'BUILD_NUMBER_TO_DESTROY', defaultValue: '', description: 'Specify the build number to destroy (only applicable if ACTION is destroy)')
    }

    environment {
        // Define environment variables
        DO_TOKEN = credentials('digitalocean') // Replace with your actual Jenkins credentials ID for DigitalOcean token
        TF_VAR_do_token = "${DO_TOKEN}"
        TF_DIR = "/home/ubuntu/Terraform-Plans" // Change this path based on your Jenkins setup
        MACHINE_NAME = "Server-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                // Checkout the code from your SCM
                checkout([
                    $class: 'GitSCM', 
                    branches: [[name: '*/main']], 
                    doGenerateSubmoduleConfigurations: false, 
                    extensions: [[$class: 'CleanCheckout']], 
                    submoduleCfg: [], 
                    userRemoteConfigs: [[credentialsId: 'GitHub', url: 'git@github.com:kumar-madhan/terraform-digitalocean.git']]
                ])
            }
        }

        stage('Terraform Init') {
            steps {
                script {
                    // Initialize Terraform
                    dir('terraform') {
                        sh 'terraform init'
                    }
                }
            }
        }

        stage('Terraform Plan') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                script {
                    // Plan Terraform changes and save the plan with build number
                    dir('terraform') {
                        sh """
                        terraform plan -var "do_token=${DO_TOKEN}" \
                              -var "Machine_Name=${MACHINE_NAME}" \
                              -out=${TF_DIR}/tfplan-${env.BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                script {
                    // Apply Terraform changes
                    dir('terraform') {
                        sh "terraform apply -input=false ${TF_DIR}/tfplan-${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' && params.BUILD_NUMBER_TO_DESTROY != '' }
            }
            steps {
                script {
                    // Destroy Terraform-managed infrastructure for the specified build number
                    dir('terraform') {
                        // Use the tfplan file to destroy the infrastructure
                        sh "terraform destroy -input=false ${TF_DIR}/tfplan-${params.BUILD_NUMBER_TO_DESTROY}"
                    }
                }
            }
        }

        stage('Ansible Playbook') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                script {
                    // Get the Droplet IP from Terraform output
                    def dropletIP = sh(script: "terraform output -raw droplet_ip", returnStdout: true).trim()

                    // Run Ansible playbook
                    dir('ansible') {
                        sh """
                        ansible-playbook -i ${dropletIP}, -u root --private-key ~/.ssh/id_rsa playbook-1.yaml
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
