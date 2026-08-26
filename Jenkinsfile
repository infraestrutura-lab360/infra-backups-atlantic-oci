pipeline {
    agent any

    // BLOCO NOVO: Cria o menu suspenso no painel do Jenkins
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
                // Aqui está o pulo do gato: puxar a chave SSH E as credenciais do S3 na mesma lista separada por vírgula
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'ssh-key-hmg-102', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                    usernamePassword(credentialsId: 'oci-s3-fluentbit-keys', usernameVariable: 'OCI_S3_ACCESS_KEY', passwordVariable: 'OCI_S3_SECRET_KEY')
                ]) {
                    sh """
                    export ANSIBLE_HOST_KEY_CHECKING=False
                    ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/deploy_fluentbit.yml \
                        -l ${params.AMBIENTE} \
                        --private-key \$SSH_KEY \
                        -u $SSH_USER \
                        -e "s3_access_key=\${OCI_S3_ACCESS_KEY} s3_secret_key=\${OCI_S3_SECRET_KEY}"
                    """
                }
            }
        }

        stage('Deploy Backup Fonte (Ansible)') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'ssh-key-hmg-102', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                    usernamePassword(credentialsId: 'oci-s3-fluentbit-keys', usernameVariable: 'OCI_S3_ACCESS_KEY', passwordVariable: 'OCI_S3_SECRET_KEY')
                ]) {
                    sh """
                    export ANSIBLE_HOST_KEY_CHECKING=False
                    ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/deploy_backup.yml \
                        -l ${params.AMBIENTE} \
                        --private-key \$SSH_KEY \
                        -u $SSH_USER \
                        -e "s3_access_key=\${OCI_S3_ACCESS_KEY} s3_secret_key=\${OCI_S3_SECRET_KEY}"
                    """
                }
            }
        }
    }
}