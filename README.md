🛡️ Projeto Infra-FluentBit & Disaster Recovery

Este repositório contém a automação de infraestrutura como código (IaC) para centralizar, padronizar e automatizar a rotação e o envio de logs, além da rotina de Disaster Recovery (Backup de Código-Fonte) para servidores hospedados na Oracle Cloud Infrastructure (OCI).

Organização: Atlantic SolutionsEscopo da Automação: Multi-Ambiente (Homologação e Produção)

📖 Sobre o Projeto

A solução foi desenhada para garantir a preservação do espaço em disco dos servidores locais, resiliência no envio de dados e uma estrutura semântica rigorosa na nuvem (OCI Object Storage). Todo o fluxo é gerenciado por uma esteira de CI/CD parametrizada, garantindo escalabilidade, controle exato de execução e versionamento.

Principais Entregas:

Log Streaming: Envio contínuo e tolerante a falhas de logs (Ruby on Rails, SO, Apache) para a OCI.

Saúde do Disco: Rotação e compactação local de logs (Logrotate).

Disaster Recovery: Backup diário e automatizado do código-fonte.

Retenção Inteligente (D-0): Manutenção da última versão do backup local para rollbacks rápidos sem onerar o disco.

🏗️ Arquitetura e Componentes

A arquitetura baseia-se no princípio de separação de responsabilidades e execução condicional:

Logrotate: Responsável exclusivo pela saúde do disco local. Rotaciona arquivos .log diariamente utilizando a diretiva copytruncate (para não interromper processos das aplicações) e mantém um histórico curto de segurança.

Fluent Bit (Otimizado para Alta Volumetria): Agente de log streaming ultraleve. Monitora arquivos em tempo real, faz buffer em disco (/var/log/fluentbit-buffer/) para evitar consumo de RAM e envia os pacotes para a OCI. Foi configurado com gatilhos dinâmicos baseados na regra "OU":  

total_file_size 500M: Corta e envia o pacote para nuvem assim que acumula 500MB de texto, prevenindo gargalos de rede em picos de acesso.  

upload_timeout 24h: Força o envio diário caso a volumetria não atinja os 500MB, reduzindo a fragmentação no Object Storage.  

Cron & Shell Script (Backup): Execução diária às 02:00 AM empacotando o código-fonte via tar e sincronizando via aws cli.  

Ansible & Jenkins: Padronização (Configuration Management) e entrega (Deployment). O playbook utiliza variáveis de perfil condicional (perfil_rails, perfil_apache) para instalar apenas o necessário em cada host.  

🗄️ Estrutura de Armazenamento (OCI S3 Compatible)O destino final dos artefatos é o OCI Object Storage (Namespace: gr7swnp1wj0e), utilizando a API compatível com Amazon S3 no bucket backups.  

A taxonomia de diretórios foi perfeitamente mapeada para alinhar todas as saídas no formato Ano/Mês/Dia, facilitando auditorias e higienização:

Logs:

/backups/aplicacoes/{nome_servidor}/logs/%Y/%m/%d/  

/backups/servidores/{nome_servidor}/apache/logs/%Y/%m/%d/ 

/backups/servidores/{nome_servidor}/so/logs/%Y/%m/%d/  
 
Código-Fonte (Backups):
 
/backups/aplicacoes/{nome_servidor}/fonte/%Y/%m/%d/  
 
🔐 Segurança e Integração Contínua (CI/CD)Governança OCI IAM
 
Para isolar o serviço de contas administrativas, recursos dedicados foram provisionados na OCI:
 
Usuário: svc-fluentbit automation 
 
  Política: plc-automacao-fluentbit (Escrita estrita no bucket correspondente). 
  
  Autenticação: Customer Secret Key (S3 Compatible).
   
Jenkins (Deploy Parametrizado)
   
  A pipeline de deploy oferece controle cirúrgico sobre a infraestrutura e protege dados sensíveis:
   
  Menu de Ambientes: O Jenkinsfile possui um bloco parameters que disponibiliza um menu de seleção (homologacao, producao, all). A escolha restringe a execução do Ansible (flag -l) exclusivamente ao grupo desejado. 
   
  Credencial oci-s3-fluentbit-keys: Injeta as chaves S3 como variáveis de ambiente no pipeline de forma segura, com escaping explícito (\$) para evitar exposição da string no console.  
   
Autenticação SSH e Inventário Dinâmico
   
  Cofre SSH: A chave privada (ssh-key-hmg-102) associada às instâncias não fica armazenada no repositório nem solta no servidor. O Jenkins gera um arquivo temporário em memória, repassa ao Ansible e o destrói imediatamente após o uso.  
   
  Inventário Modular (hosts.ini): Os servidores são classificados por grupo e marcados com booleanos que definem o seu comportamento durante o provisionamento:

  [homologacao]
  hmg-col-102 ansible_host=172.33.18.109 ansible_user=ubuntu nome_servidor=hmg-col-102 perfil_rails=true perfil_apache=true

  [producao]
  #prd-tst-101 ansible_host=00.0.0.0 ansible_user=ubuntu nome_servidor=prd-tst-101 perfil_rails=false perfil_apache=true

🔄 Rotina de Backup e Retenção de Fonte 

A role do Ansible backup_fonte implementa o backup automatizado da aplicação.  

Diretório Alvo: /home/ubuntu/AdAlive-Apps/AdAlive-doTerrahmlg (Ignorando pastas transitórias como log/, tmp/ e node_modules/).  

Retenção Nuvem: Versionamento diário contínuo no bucket.

Retenção Local (D-0): Apenas o backup da madrugada atual é mantido no servidor. O script utiliza o comando find ... ! -name ... -delete para expurgar automaticamente arquivos antigos, protegendo o disco.  

Padrão de Saída: AdAlive-doTerrahmlg_YYYY-MM-DD.tar.gz

🚑 Runbook: Restauração de Emergência (Rollback Local)

Em caso de falha crítica na aplicação onde seja necessário voltar o código para o estado da última madrugada, utilize o backup local D-0 

1. Acesse o diretório base da aplicação
cd /home/ubuntu/AdAlive-Apps/

# 2. Descompacte o último backup gerado (substitua a data no nome do arquivo)
tar -xzvf AdAlive-doTerrahmlg_YYYY-MM-DD.tar.gz

# 3. Reinicie os serviços da aplicação (exemplo de comando)
sudo systemctl restart adalive-service...