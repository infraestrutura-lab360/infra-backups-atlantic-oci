pipeline {
    agent any
    
    environment {
        // O Jenkins vai puxar a credencial 'oci-s3-fluentbit-keys' 
        // e separar automaticamente em USR (Access Key) e PSW (Secret Key)
        OCI_S3_CREDS = credentials('oci-s3-fluentbit-keys')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Deploy Fluent Bit (Ansible)') {
            steps {
                // Aqui executamos o Ansible passando as chaves da OCI como variáveis extras (-e)
                sh '''
                export ANSIBLE_HOST_KEY_CHECKING=False
                ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/deploy_fluentbit.yml \
                -e "s3_access_key=${OCI_S3_CREDS_USR} s3_secret_key=${OCI_S3_CREDS_PSW}"
                '''
            }
        }
    }
}