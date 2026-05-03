pipeline {
    agent any

    parameters {
        choice(name: 'action', choices: ['apply', 'destroy'], description: 'Terraform Action')
    }

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        TF_IN_AUTOMATION   = 'true'
    }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/ShrinivasC96/Terraform-monitoring.git'
                sh 'ls -l'
            }
        }

        stage('Configure AWS CLI') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'AWS_Access_Key',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region $AWS_DEFAULT_REGION
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Terraform Init') {
            steps {
                sh 'terraform init'
            }
        }

        stage('Terraform Format Check') {
            steps {
                sh 'terraform fmt -check -recursive'
            }
        }

        stage('Terraform Validate') {
            steps {
                sh 'terraform validate'
            }
        }

        stage('Terraform Plan') {
            when { expression { params.action == 'apply' } }
            steps {
                script {
                    def rc = sh(script: "terraform plan -detailed-exitcode -out=tfplan", returnStatus: true)
                    if (rc == 0) {
                        echo "No infrastructure changes detected"
                        env.TF_CHANGES = "false"
                    } else if (rc == 2) {
                        echo "Infrastructure changes detected"
                        env.TF_CHANGES = "true"
                    } else {
                        error "Terraform plan failed"
                    }
                }
            }
        }

        stage('Approval Before Apply') {
            when {
                allOf {
                    expression { params.action == 'apply' }
                    expression { env.TF_CHANGES == "true" }
                }
            }
            steps {
                input message: "Review the plan above. Approve Terraform Apply?"
            }
        }

        stage('Terraform Apply') {
            when {
                allOf {
                    expression { params.action == 'apply' }
                    expression { env.TF_CHANGES == "true" }
                }
            }
            steps {
                sh 'terraform apply --auto-approve tfplan'
            }
        }

        stage('Approval Before Destroy') {
            when { expression { params.action == 'destroy' } }
            steps {
                input message: "WARNING: This will destroy all resources. Are you sure?"
            }
        }

        stage('Terraform Destroy') {
            when { expression { params.action == 'destroy' } }
            steps {
                sh 'terraform destroy --auto-approve'
            }
        }

        stage('Configure EKS') {
            when {
                allOf {
                    expression { params.action == 'apply' }
                    expression { env.TF_CHANGES == "true" }
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'AWS_Access_Key',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        CALLER=$(aws sts get-caller-identity --query Arn --output text)
                        echo "Logged in as: $CALLER"

                        aws eks create-access-entry \
                          --cluster-name my-eks-cluster \
                          --principal-arn $CALLER \
                          --type STANDARD \
                          --region $AWS_DEFAULT_REGION || echo "Access entry may already exist"

                        aws eks associate-access-policy \
                          --cluster-name my-eks-cluster \
                          --principal-arn $CALLER \
                          --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
                          --access-scope \'{"type":"cluster"}\' \
                          --region $AWS_DEFAULT_REGION || echo "Policy may already be associated"

                        aws eks update-kubeconfig \
                          --name my-eks-cluster \
                          --region $AWS_DEFAULT_REGION \
                          --alias my-eks-cluster

                        sudo mkdir -p /var/lib/jenkins/.kube
                        sudo cp ~/.kube/config /var/lib/jenkins/.kube/config
                        sudo chown -R jenkins:jenkins /var/lib/jenkins/.kube/
                        sudo chmod 600 /var/lib/jenkins/.kube/config

                        echo "Waiting 30s for nodes to be ready..."
                        sleep 30
                        kubectl get nodes
                    '''
                }
            }
        }
    }

    post {
        failure {
            echo "Pipeline failed — no infrastructure changes were made"
        }
        success {
            echo "Pipeline completed successfully...!"
        }
    }
}
