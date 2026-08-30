pipeline {
    agent any

    parameters {
        choice(name: 'ALVO', choices: ['homologacao', 'prd-nov-101', 'prd-nov-102'], description: 'Escolha o ambiente ou servidor alvo para o deploy')
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
                    def chaveSSH
                    def limiteAnsible = params.ALVO 
                    
                    switch(params.ALVO) {
                        case 'prd-nov-101':
                            chaveSSH = 'ssh-key-prd-nov-101'
                            break
                        case 'prd-nov-102':
                            chaveSSH = 'ssh-key-prd-nov-102'
                            break
                        case 'homologacao':
                            chaveSSH = 'ssh-key-hmg-102'
                            break
                        default:
                            error("Alvo não reconhecido: ${params.ALVO}")
                    }
                    
                    withCredentials([
                        sshUserPrivateKey(credentialsId: chaveSSH, keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
                        usernamePassword(credentialsId: 'oci-atlantic-s3-backups-keys', usernameVariable: 'OCI_S3_ACCESS_KEY', passwordVariable: 'OCI_S3_SECRET_KEY')
                    ]) {
                        sh """
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/deploy_backup.yml \
                            -l ${limiteAnsible} \
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