pipeline {
    // Define que o pipeline pode rodar em qualquer nó do Jenkins disponível
    agent any

    // Opcional: Define variáveis de ambiente para facilitar a manutenção
    environment {
        ANSIBLE_CONFIG = "${WORKSPACE}/ansible/ansible.cfg" // Caso você crie um futuramente
    }

    stages {
        stage('Checkout Código') {
            steps {
                // Baixa a versão mais recente do código do seu repositório Git
                checkout scm
            }
        }

        stage('Validação de Sintaxe (Lint)') {
            steps {
                // A instrução dir() muda o contexto de execução para a pasta 'ansible'
                dir('ansible') {
                    // Valida se o playbook não tem erros de sintaxe antes de aplicar
                    sh 'ansible-playbook -i inventories/hosts.ini playbooks/deploy_fluentbit.yml --syntax-check'
                }
            }
        }

        stage('Deploy Fluent Bit') {
            steps {
                // Chama a credencial que configuramos no Jenkins (oci-s3-fluentbit-keys)
                // NUNCA coloque as chaves hardcoded no código.
                withCredentials([usernamePassword(credentialsId: 'oci-s3-fluentbit-keys', passwordVariable: 'SECRET_KEY', usernameVariable: 'ACCESS_KEY')]) {
                    
                    // Entra na pasta ansible novamente para executar o playbook
                    dir('ansible') {
                        sh '''
                            ansible-playbook -i inventories/hosts.ini playbooks/deploy_fluentbit.yml \
                            --extra-vars "s3_access_key=${ACCESS_KEY} s3_secret_key=${SECRET_KEY}"
                        '''
                    }
                }
            }
        }
    }
    
    // Ações a serem tomadas após a execução do pipeline (opcional)
    post {
        always {
            cleanWs() // Limpa o workspace do Jenkins após o uso
        }
        success {
            echo 'Deploy do Fluent Bit realizado com sucesso!'
        }
        failure {
            echo 'O deploy falhou. Verifique os logs do Ansible acima.'
        }
    }
}