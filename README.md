# 🛡️ Projeto Infra-Backups OCI & Retenção de Logs

Este repositório contém a automação de infraestrutura como código (IaC) para centralizar, padronizar e automatizar a rotação e o envio de logs consolidados (D-1), além da rotina de Disaster Recovery (Backup de Código-Fonte, pacotes WAR e microsserviços JAR) para servidores hospedados na Oracle Cloud Infrastructure (OCI).

**Organização:** Empresa de Tecnologia (Omitido por confidencialidade)  
**Escopo da Automação:** Multi-Ambiente (Homologação e Produção)

## 📖 Sobre o Projeto
A solução foi desenhada para garantir a preservação do espaço em disco dos servidores locais, resiliência no envio de dados e uma estrutura semântica rigorosa na nuvem (OCI Object Storage). Todo o fluxo é gerenciado por uma esteira de CI/CD parametrizada sob o conceito de *Zero Trust*, garantindo escalabilidade, controle exato de execução e isolamento de acessos.

**Principais Entregas:**
* **Sem Agentes (Agentless):** Remoção de daemons pesados (antigo Fluent Bit). A arquitetura agora usa ferramentas nativas do Linux, eliminando o consumo excessivo de CPU e RAM.
* **Retenção Consolidada (D-1):** Envio diário de logs unificados (Rails, Tomcat e Quarkus), acabando com a fragmentação de arquivos na nuvem.
* **Disaster Recovery:** Backup diário automatizado do código-fonte (Rails), pacotes compilados (Tomcat/WAR) e microsserviços Java (Quarkus/JAR).
* **Limpeza Automática:** Manutenção inteligente local para não onerar o disco do servidor (Retenção D-0 local).

## 🏗️ Arquitetura e Componentes
A arquitetura baseia-se na separação de responsabilidades e janelas de execução para não sufocar o servidor:

* **Logrotate Local:** Responsável exclusivo pela saúde do disco local. Rotaciona arquivos `.log` diariamente à meia-noite utilizando a diretiva `copytruncate` (para não interromper processos das aplicações) e injeta a diretiva `dateyesterday` para sincronia de dados em serviços nativos.
* **Cron & Bash Script (Logs D-1):** Executado diariamente às **00:01 AM**. Varre o servidor em busca de logs fechados do dia anterior (SO, Apache, Rails, Tomcat, Quarkus), os compacta de forma segura no `/tmp` e envia para a nuvem como um arquivo único datado.
* **Cron & Bash Script (Backup Fonte/WAR/JAR):** Executado diariamente às **02:00 AM**. Empacota o código-fonte (excluindo pastas inúteis como logs e `node_modules`), pacotes `.war` e backups gerados por deploy de `.jar`, envia via `aws cli` e destrói backups locais antigos.
* **Ansible:** Padronização (Configuration Management) baseada em `host_vars`. Cria a automação dinamicamente com base nas aplicações declaradas (se é Rails, Tomcat ou Quarkus; Fonte, WAR ou JAR) para cada host.

## 🗄️ Estrutura de Armazenamento (OCI S3 Compatible)
O destino final dos artefatos é o OCI Object Storage, utilizando a API compatível com Amazon S3 no bucket `backups`. A taxonomia de diretórios foi mapeada para alinhar as saídas no formato `Ano/Mês/Dia`:

**Logs:**
* `/backups/aplicacoes/{nome_servidor}/{id_bucket}/logs/%Y/%m/%d/`
* `/backups/servidores/{nome_servidor}/apache/logs/%Y/%m/%d/`
* `/backups/servidores/{nome_servidor}/so/logs/%Y/%m/%d/`

**Disaster Recovery (Código e Compilados):**
* `/backups/aplicacoes/{nome_servidor}/{id_bucket}/fonte/%Y/%m/%d/` (Usado para código Rails e arquivos JAR)
* `/backups/aplicacoes/{nome_servidor}/{id_bucket}/war/%Y/%m/%d/`

## 🔐 Segurança e Integração Contínua (CI/CD)

### Governança OCI IAM
Para isolar o serviço de contas administrativas, recursos dedicados foram provisionados na OCI:
* **Usuário:** `svc-automacao-backups-oci`
* **Política:** `plc-automacao-backups-oci` (Escrita estrita no bucket correspondente).
* **Autenticação:** Customer Secret Key (S3 Compatible).

### Jenkins (Deploy Parametrizado e Zero Trust)
A pipeline de deploy oferece controle cirúrgico e protege a infraestrutura contra execuções acidentais em massa:
* **Menu de Alvos Restritos:** O Jenkinsfile possui um bloco `parameters` (`ALVO`) que obriga o operador a escolher um servidor específico por vez (ex: `prd-tst-101`). A escolha restringe a execução do Ansible (`-l prd-tst-101`) exclusivamente àquela máquina.
* **Isolamento de Chaves SSH:** Não há compartilhamento de credenciais. Através de um bloco `switch` no Groovy, o Jenkins injeta temporariamente apenas a chave SSH pertencente àquele servidor específico, impedindo movimentação lateral.
* **Proteção de Credenciais:** A credencial OCI S3 é injetada na memória, com escaping explícito (`\$`) para evitar exposição no log do console.

### Inventário Modular (Ansible)
A declaração de regras é feita de forma granular usando o inventário e os arquivos de variáveis de host (`host_vars`):

**Exemplo de `hosts.ini`:**
```ini
[homologacao]
hmg-tst-101 ansible_host=10.0.0.100 ansible_user=ubuntu nome_servidor=hmg-tst-101

[producao]
prd-tst-101 ansible_host=10.0.0.101 ansible_user=ubuntu nome_servidor=prd-tst-101
Exemplo de host_vars/prd-tst-101.yml (Aplicações Multi-Stack):

YAML
---
perfil_apache: true

aplicacoes:
  - id_bucket: "meu-app-rails"
    nome_pasta: "MeuApp-Rails"
    tipo_app: "rails"
    tipo_backup: "fonte"
    log_principal: "rails_server_app.log"

  - id_bucket: "meu-app-quarkus"
    caminho_base: "/u01/microservices"
    nome_pasta: "" 
    tipo_app: "quarkus"
    tipo_backup: "jar"
    log_principal: "ms-app-service.log"
    caminho_backup_local: "/u01/deployments/backups/microservices"
🚑 Runbook: Restauração de Emergência (Rollback Local)
Em caso de falha crítica na aplicação onde seja necessário voltar o código para o estado da última madrugada, utilize o último artefato compactado gerado pela rotina D-0.

Bash
# 1. Acesse o diretório base da aplicação (Exemplo Rails)
cd /home/ubuntu/MeuApp-Rails/

# 2. Descompacte o último backup gerado (substitua a data no nome do arquivo)
tar -xzvf Nome-Pasta.YYYY-MM-DD.tar.gz

# (Para microsserviços JAR, basta descompactar e substituir o artefato .jar na pasta /u01/microservices/)

# 3. Reinicie os serviços da aplicação
sudo systemctl restart puma  # ou tomcat9 / app-service
