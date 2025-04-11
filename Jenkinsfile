pipeline {
    agent {
        docker {
            image 'jenkins/jenkins:lts'  
            args '-v /var/jenkins_home:/var/jenkins_home'  
        }
    }
    environment {
        AWS_ACCESS_KEY_ID = credentials('aws-credentials')
        AWS_SECRET_ACCESS_KEY = credentials('aws-credentials')
    }
    stages {
        stage('Install Terraform') {
            steps {
                script {
                    // تثبيت Terraform داخل الحاوية
                    sh '''
                    # تحديث الحاوية وتثبيت الأدوات المطلوبة
                    apt-get update
                    apt-get install -y wget unzip

                    # تحميل وتثبيت Terraform
                    wget https://releases.hashicorp.com/terraform/1.11.4/terraform_1.11.4_linux_amd64.zip
                    unzip terraform_1.11.4_linux_amd64.zip
                    sudo mv terraform /usr/local/bin/

                    # تأكيد أن Terraform تم تثبيته بنجاح
                    terraform --version
                    '''
                }
            }
        }
        stage('Terraform Init') {
            steps {
                dir('/var/jenkins_home/terraform-aws-ec2') {
                    sh 'terraform init'
                }
            }
        }
        stage('Terraform Apply') {
            steps {
                dir('/var/jenkins_home/terraform-aws-ec2') {
                    sh 'terraform apply -auto-approve'
                }
            }
        }
        stage('Generate Ansible Inventory') {
            steps {
                dir('/var/jenkins_home/terraform-aws-ec2') {
                    sh '''
                    terraform output -raw ec2_public_ip > ansible/inventory
                    echo "[ec2]" > ansible/hosts
                    echo "$(cat ansible/inventory)" >> ansible/hosts
                    '''
                }
            }
        }
        stage('Run Ansible Playbook') {
            agent {
                docker {
                    image 'ansible/ansible-runner'
                    args '-v /var/jenkins_home/terraform-aws-ec2/ansible:/ansible'
                }
            }
            steps {
                dir('/var/jenkins_home/terraform-aws-ec2/ansible') {
                    sh '''
                    ansible-playbook -i hosts deploy.yml -u ec2-user --private-key /ansible/my-key.pem
                    '''
                }
            }
        }
    }
    post {
        always {
            dir('/var/jenkins_home/terraform-aws-ec2') {
                sh 'terraform destroy -auto-approve || true'
            }
        }
    }
}
