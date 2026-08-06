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
        withCredentials([
            sshUserPrivateKey(credentialsId: 'ssh-key-hmg-102', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')
        ]) {
            sh '''
            export ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/deploy_fluentbit.yml \
                --private-key $SSH_KEY \
                -u $SSH_USER \
                -e "s3_access_key=${OCI_S3_ACCESS_KEY} s3_secret_key=${OCI_S3_SECRET_KEY}"
            '''
        }
    }
}

        stage('Deploy Backup Fonte (Ansible)') {
    steps {
        withCredentials([
            sshUserPrivateKey(credentialsId: 'ssh-key-hmg-102', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')
        ]) {
            sh '''
            export ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/deploy_backup.yml \
                --private-key $SSH_KEY \
                -u $SSH_USER \
                -e "s3_access_key=${OCI_S3_ACCESS_KEY} s3_secret_key=${OCI_S3_SECRET_KEY}"
            '''
        }
    }
}
    }
}