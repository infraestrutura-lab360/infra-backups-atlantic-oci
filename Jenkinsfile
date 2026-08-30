pipeline {
    agent any

    parameters {
        choice(name: 'AMBIENTE', choices: ['homologacao', 'producao', 'all'], description: 'Escolha o ambiente para o deploy')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Deploy Backup Fonte e Logs (Ansible)') {
            steps {
                script {
                    def chaveSSH = (params.AMBIENTE == 'producao') ? 'ssh-key-prd-nov-102' : 'ssh-key-hmg-102'
                    
                    withCredentials([
                        sshUserPrivateKey(credentialsId: chaveSSH, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                        usernamePassword(credentialsId: 'oci-atlantic-s3-backups-keys', usernameVariable: 'OCI_S3_ACCESS_KEY', passwordVariable: 'OCI_S3_SECRET_KEY')
                    ]) {
                        sh """
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/deploy_backup.yml \
                            -l ${params.AMBIENTE} \
                            --private-key \$SSH_KEY \
                            -u \$SSH_USER \
                            -e "s3_access_key=\${OCI_S3_ACCESS_KEY} s3_secret_key=\${OCI_S3_SECRET_KEY}"
                        """
                    }
                }
            }
        }
    }
}