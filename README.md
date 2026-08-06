# 🛡️ Projeto Infra-FluentBit & Disaster Recovery

Este repositório contém a automação de infraestrutura como código (IaC) para centralizar, padronizar e automatizar a rotação e o envio de logs, além da rotina de Disaster Recovery (Backup de Código-Fonte) para servidores hospedados na Oracle Cloud Infrastructure (OCI).

**Servidor Alvo Inicial:** `hmg-col-102` (`172.33.18.109`)
**Organização:** Atlantic Solutions

---

## 📖 Sobre o Projeto

A solução foi desenhada para garantir a preservação do espaço em disco dos servidores locais, resiliência no envio de dados e uma estrutura semântica na nuvem (OCI Object Storage). Todo o fluxo é gerenciado por uma esteira de CI/CD, garantindo rigoroso controle de acesso e versionamento.

### Principais Entregas:
1. **Log Streaming:** Envio contínuo de logs (Ruby on Rails, SO, Apache) para a OCI.
2. **Saúde do Disco:** Rotação e compactação local de logs.
3. **Disaster Recovery:** Backup diário e automatizado do código-fonte.
4. **Retenção Inteligente (D-0):** Manutenção da última versão do backup local para *rollbacks* rápidos.

---

## 🏗️ Arquitetura e Componentes

A arquitetura baseia-se no princípio de separação de responsabilidades:

* **Logrotate:** Responsável exclusivo pela saúde do disco local. Rotaciona arquivos `.log` diariamente utilizando a diretiva `copytruncate` (para não interromper processos Rails) e mantém um histórico curto (ex: 7 dias).
* **Fluent Bit:** Agente de log streaming ultraleve. Monitora arquivos em tempo real, realiza buffer em disco (`/var/log/fluentbit-buffer/`) para tolerância a falhas e envia os pacotes para a OCI (parâmetros de `upload_timeout 1h` e `total_file_size 200M`).
* **Cron & Shell Script (Backup):** Execução diária às `02:00 AM` empacotando o código-fonte via `tar` e sincronizando via `aws cli`.
* **Ansible & Jenkins:** Padronização (Configuration Management) e entrega (Deployment) de toda a configuração.

---

## 🗄️ Estrutura de Armazenamento (OCI S3 Compatible)

O destino final dos artefatos é o **OCI Object Storage** (Namespace: `gr7swnp1wj0e`), utilizando a API compatível com Amazon S3 no bucket `backups`.

A taxonomia de diretórios foi mapeada para organizar as aplicações de forma semântica e escalável:

**Logs:**
* `/backups/aplicacoes/hmg-col-102/logs/ano/mes/`
* `/backups/servidores/hmg-col-102/apache/logs/ano/mes/`
* `/backups/servidores/hmg-col-102/so/logs/ano/mes/`

**Código-Fonte (Backups):**
* `/backups/aplicacoes/hmg-col-102/fonte/ano/mes/dia/`

---

## 🔐 Segurança e Integração Contínua (CI/CD)

### IAM e Oracle Cloud
Para isolar o serviço de contas administrativas, recursos dedicados foram provisionados na OCI:
* **Usuário:** `svc-fluentbit automation`
* **Política:** `plc-automacao-fluentbit` (Escrita estrita no bucket correspondente).
* **Autenticação:** Customer Secret Key (S3 Compatible).

### Jenkins
O deploy automatizado protege dados sensíveis e gerencia o repositório utilizando:
* **Credencial `oci-s3-fluentbit-keys`:** Injeta as chaves S3 como variáveis de ambiente (`$OCI_S3_CREDS`) no *pipeline*.
* **Token `deploy-fluentbit-oci-atlantic`:** Token de serviço vitalício e exclusivo para o checkout do repositório via Jenkins.

---

## 🔄 Rotina de Backup e Retenção de Fonte

A *role* do Ansible `backup_fonte` implementa o backup automatizado da aplicação `AdAlive-doTerrahmlg`. 

* **Diretório Alvo:** `/home/ubuntu/AdAlive-Apps/AdAlive-doTerrahmlg` (Ignorando pastas transitórias como `log/`, `tmp/` e `node_modules/`).
* **Retenção Nuvem:** Versionamento diário contínuo no bucket.
* **Retenção Local (D-0):** Apenas o backup da madrugada atual é mantido no servidor (em `/home/ubuntu/AdAlive-Apps/`). O script utiliza o comando `find ... ! -name ... -delete` para expurgar automaticamente arquivos antigos, protegendo o disco.
* **Padrão de Saída:** `AdAlive-doTerrahmlg_YYYY-MM-DD.tar.gz`

---

## 🚑 Runbook: Restauração de Emergência (Rollback Local)

Em caso de falha crítica na aplicação onde seja necessário voltar o código para o estado da última madrugada, utilize o backup local D-0 (Botão de Pânico):

```bash
# 1. Acesse o diretório base da aplicação
cd /home/ubuntu/AdAlive-Apps/

# 2. Descompacte o último backup gerado (substitua a data no nome do arquivo)
tar -xzvf AdAlive-doTerrahmlg_YYYY-MM-DD.tar.gz

# 3. Reinicie os serviços da aplicação (exemplo de comando)
sudo systemctl restart adalive-server
