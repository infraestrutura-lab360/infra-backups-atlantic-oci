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
        
        stage('Deploy Fluent Bit (Ansible)') {
            steps {
                script {
                    // Lógica para selecionar a chave correta com base no ambiente escolhido
                    def chaveSSH = (params.AMBIENTE == 'producao') ? 'ssh-key-prd-nov-102' : 'ssh-key-hmg-102'
                    
                    withCredentials([
                        sshUserPrivateKey(credentialsId: chaveSSH, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                        usernamePassword(credentialsId: 'oci-s3-fluentbit-keys', usernameVariable: 'OCI_S3_ACCESS_KEY', passwordVariable: 'OCI_S3_SECRET_KEY')
                    ]) {
                        sh """
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/deploy_fluentbit.yml \
                            -l ${params.AMBIENTE} \
                            --private-key \$SSH_KEY \
                            -u \$SSH_USER \
                            -e "s3_access_key=\${OCI_S3_ACCESS_KEY} s3_secret_key=\${OCI_S3_SECRET_KEY}"
                        """
                    }
                }
            }
        }

        stage('Deploy Backup Fonte (Ansible)') {
            steps {
                script {
                    // Aplica a mesma lógica para o backup
                    def chaveSSH = (params.AMBIENTE == 'producao') ? 'ssh-key-prd-nov-102' : 'ssh-key-hmg-102'
                    
                    withCredentials([
                        sshUserPrivateKey(credentialsId: chaveSSH, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                        usernamePassword(credentialsId: 'oci-s3-fluentbit-keys', usernameVariable: 'OCI_S3_ACCESS_KEY', passwordVariable: 'OCI_S3_SECRET_KEY')
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