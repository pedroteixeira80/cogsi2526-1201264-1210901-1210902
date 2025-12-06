# CA6 - Pipelines CI/CD

## Índice
1. [Pipeline CI/CD para Implementação de Serviços REST com Spring e implementação numa máquina virtual local](#pipeline-cicd-para-implementação-de-serviços-rest-com-spring-e-implementação-numa-máquina-virtual-local)
2. [Pipeline CI/CD para Implementação de uma Aplicação Spring com Docker](#pipeline-cicd-para-implementação-de-uma-aplicação-spring-com-docker)
    1. [Automatização da Configuração da Infraestrutura](#automatização-da-configuração-da-infraestrutura)
        1. [Passo 1: Criar uma VM de Produção](#passo-1-criar-uma-vm-de-produção)
        2. [Passo 2: Provisionar a VM com Ansible](#passo-2-provisionar-a-vm-com-ansible)
    2. [Configuração do Pipeline CI/CD com Jenkins](#configuração-do-pipeline-cicd-com-jenkins)
        1. [Executar Jenkins Localmente](#executar-jenkins-localmente)
        2. [Definir a Lógica do Pipeline no `Jenkinsfile`](#definir-a-lógica-do-pipeline-no-jenkinsfile)
            1. [Etapa 1: Checkout](#etapa-1-checkout)
            2. [Etapa 2: Assemble](#etapa-2-assemble)
            3. [Etapa 3: Test](#etapa-3-test)
            4. [Etapa 4: Tag Docker Image](#etapa-4-tag-docker-image)
            5. [Etapa 5: Archive](#etapa-5-archive)
            6. [Etapa 6: Push Docker Image](#etapa-6-push-docker-image)
            7. [Etapa 7: Deploy](#etapa-7-deploy) 
       3. [Configurar Notificações e Verificação de Implementação](#configurar-notificações-e-verificação-de-implementação)
            1. [Passo 1: Notificações](#passo-1-notificações)
            2. [Passo 2: Verificação de Implementação](#passo-2-verificação-de-implementação)
       4. [Execução e Análise do Jenkinsfile](#execução-e-análise-do-jenkinsfile)
       5. [Problemas Comuns](#problemas-comuns)
3. [Alternativa ao Jenkins](#alternativa-ao-jenkins)

## Pipeline CI/CD para Implementação de Serviços REST com Spring e implementação numa máquina virtual local

1. `Vagrantfile`

   O Vagrantfile define e configura duas máquinas virtuais (VMs) denominadas blue e green usando o Vagrant. Estas VMs são criadas com a imagem base ubuntu/bionic64 e provisionadas usando Ansible. Este ficheiro gere essencialmente a criação e configuração das VMs que fazem parte dos seus ambientes de desenvolvimento e implementação.

```ruby
Vagrant.configure("2") do |config|
  # Configuração padrão para ambas as VMs
  config.vm.box = "ubuntu/bionic64"

  # VM BLUE
  config.vm.define "blue" do |blue|
    blue.vm.network "private_network", ip: "192.168.56.10"
    blue.vm.network "forwarded_port", guest: 8080, host: 8081

    # Provisionamento com Ansible
    blue.vm.provision "ansible" do |ansible|
      ansible.playbook = "blue.yml"
      ansible.inventory_path = "./hosts.ini"
    end

    blue.vm.provider "virtualbox" do |vb|
      vb.name = "VM-Blue"
      vb.memory = "2048"
      vb.cpus = 2
    end
  end

  # VM GREEN
  config.vm.define "green" do |green|
    green.vm.network "private_network", ip: "192.168.56.11"
    green.vm.network "forwarded_port", guest: 8080, host: 8082

    # Provisionamento com Ansible
    green.vm.provision "ansible" do |ansible|
      ansible.playbook = "green.yml"
      ansible.inventory_path = "./hosts.ini"
    end

    green.vm.provider "virtualbox" do |vb|
      vb.name = "VM-Green"
      vb.memory = "2048"
      vb.cpus = 2
    end
  end
end
```

**Secções e Funcionalidades Principais:**

- Configuração Base:

`config.vm.box = "ubuntu/bionic64"` especifica que as VMs irão usar a box base Ubuntu 18.04 (Bionic).


- Configuração da VM Blue:

`blue.vm.network "private_network", ip: "192.168.56.10"` configura uma rede privada para a VM blue com o IP 192.168.56.10.

`blue.vm.network "forwarded_port", guest: 8080, host: 8081` encaminha a porta 8080 dentro da VM para a porta 8081 na máquina host, permitindo acesso externo.

Provisionamento: A VM é provisionada usando o playbook Ansible `blue.yml`, e o ficheiro de inventário `hosts.ini` é usado para definir os hosts.

`blue.vm.provider "virtualbox"` configura os recursos da VM, alocando 2GB de RAM e 2 CPUs.


- Configuração da VM Green:

Semelhante à VM blue, a VM green é atribuída com um IP privado (`192.168.56.11`) e o encaminhamento de portas é configurado para a porta 8080 (host 8082).

O provisionamento é feito usando o playbook Ansible `green.yml`.

2. `hosts.ini`

O ficheiro `hosts.ini` é usado pelo Ansible para especificar o inventário dos hosts (VMs) a serem geridos. Define os endereços IP das VMs blue e green, juntamente com as credenciais SSH necessárias para se conectar a elas.

```ini
[blue]
192.168.56.10 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/blue/virtualbox/private_key

[green]
192.168.56.11 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/green/virtualbox/private_key
```

- [blue]: Especifica a VM blue com o seu endereço IP e credenciais SSH. O Ansible usará a chave privada para autenticar ao conectar-se à VM.

- [green]: De forma semelhante, isto define a VM green e as suas credenciais SSH.

3. `blue.yml`

O ficheiro `blue.yml` é um playbook Ansible para provisionar a VM blue. Automatiza tarefas como atualizações de sistema, instalação de pacotes, clonagem de repositório, compilação da aplicação e inicialização da aplicação.

**Secções e Funcionalidades Principais:**

- Atualizar e fazer upgrade dos pacotes do sistema: Garante que o sistema está atualizado.
```yaml
- name: Atualizar e fazer upgrade dos pacotes do sistema
      apt:
        update_cache: yes
        upgrade: dist
```

- Instalar pacotes necessários: Instala dependências como Git, OpenJDK 17, Gradle e utilitários.
```yaml
- name: Instalar pacotes necessários
      apt:
        name:
          - git
          - openjdk-17-jdk
          - unzip
          - wget
        state: present
```

- Instalar Gradle: Descarrega e instala a ferramenta de compilação Gradle.
```yaml
- name: Instalar Gradle
      shell: |
        wget https://services.gradle.org/distributions/gradle-7.6-bin.zip -P /tmp
        sudo unzip -d /opt/gradle /tmp/gradle-7.6-bin.zip
        sudo ln -s /opt/gradle/gradle-7.6/bin/gradle /usr/bin/gradle
```

- Clonar repositório: Clona um repositório GitHub específico contendo o código do projeto.
```yaml
- name: Clonar repositório
  git:
    repo: "https://github.com/your/repo.git"
    dest: /home/vagrant/project
```

- Compilar a aplicação Gradle: Compila o projeto usando Gradle.
```yaml
- name: Compilar a aplicação Gradle
  shell: |
    cd /home/vagrant/project/CA2
    ./gradlew build
```

- Iniciar a aplicação: Inicia a aplicação em segundo plano usando Java.
```yaml
- name: Iniciar a aplicação
  shell: |
    nohup java -jar /home/vagrant/project/CA2/build/libs/*.jar > /dev/null 2>&1 &
```

4. `green.yml`

O playbook `green.yml` é muito semelhante ao blue.yml. Provisiona a VM green com o mesmo conjunto de dependências e configuração, garantindo que ambas as VMs têm as ferramentas necessárias para compilar e executar a aplicação.

```yaml
---
- name: Provisionar VM Green
  hosts: green
  tasks:
    - name: Atualizar e fazer upgrade dos pacotes do sistema
      apt:
        update_cache: yes
        upgrade: dist

    - name: Instalar pacotes necessários
      apt:
        name:
          - git
          - openjdk-17-jdk
          - unzip
          - wget
        state: present

    - name: Instalar Gradle
      shell: |
        wget https://services.gradle.org/distributions/gradle-7.6-bin.zip -P /tmp
        sudo unzip -d /opt/gradle /tmp/gradle-7.6-bin.zip
        sudo ln -s /opt/gradle/gradle-7.6/bin/gradle /usr/bin/gradle
```

**Secções e Funcionalidades Principais:**
- Inclui as mesmas tarefas que em blue.yml:
  - Atualização do sistema
  - Instalação de Git, OpenJDK, Gradle, etc.
  - Instalação do Gradle através de um comando shell.

5. `Jenkinsfile`

O `Jenkinsfile` é usado pelo Jenkins para definir um pipeline CI/CD. Automatiza o processo de compilação, teste, arquivo e implementação da aplicação.

**Secções e Funcionalidades Principais:**

- Pipeline

```groovy
pipeline {
    agent any
    ...
}
```
`agent any`: Isto diz ao Jenkins para executar o pipeline em qualquer executor ou máquina disponível. Isto significa que o pipeline pode ser executado em qualquer nó Jenkins disponível, não restrito a nenhum em particular.

- Etapa Checkout

```groovy
stage('Checkout') {
    steps {
        git branch: 'main',
            url: 'https://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@github.com/pedroteixeira80/cogsi2526-1201264-1210901-1210902.git'
    }
}
```

`Checkout`: Esta etapa obtém a versão mais recente do código do ramo main do repositório GitHub especificado. Garante que o Jenkins tem o código mais recente para trabalhar.

`git branch`: O ramo especifica o ramo Git a fazer checkout, que neste caso é main.

`url`: O URL do repositório Git é usado para clonar o repositório. O URL contém um token de autenticação para acesso ao GitHub.

- Etapa Assemble

```groovy
stage('Assemble') {
    steps {
        dir('CA3') {
            // Executar compilação Gradle dentro do diretório 'CA3'
            sh './gradlew clean build --no-daemon'
        }
    }
}
```

`Assemble`: Esta etapa é responsável por compilar a aplicação usando Gradle.

`dir('CA3')`: O bloco dir diz ao Jenkins para mudar o diretório de trabalho para CA3. Isto garante que os comandos Gradle são executados dentro do diretório CA3, onde residem os seus ficheiros Gradle e código fonte do projeto.

`sh './gradlew clean build --no-daemon'`: Isto executa o comando de compilação Gradle. Limpa as saídas de compilação anteriores e gera novas. A flag --no-daemon garante que o Gradle é executado sem o daemon em segundo plano, o que é útil para ambientes CI para evitar processos residuais.

- Etapa Test

```groovy
stage('Test') {
    steps {
        dir('CA3') {
            // Executar testes Gradle dentro do diretório 'CA3'
            sh './gradlew test --no-daemon'
        }
    }
    post {
        always {
            junit 'CA3/build/test-results/**/*.xml'
        }
    }
}
```
`Test`: Esta etapa executa testes unitários usando Gradle.

`dir('CA3')`: O bloco dir garante novamente que os testes são executados dentro do diretório CA3.

`sh './gradlew test --no-daemon'`: Este comando aciona a execução de testes Gradle. Executa todos os testes definidos no projeto usando Gradle.

`post always`: O bloco post é executado após a conclusão da etapa Test, independentemente de os testes terem passado ou falhado. Aqui, é usado para processar os resultados dos testes.

`junit 'CA3/build/test-results/**/*.xml'`: Isto recolhe e processa ficheiros de resultados de testes JUnit. Procura ficheiros .xml no diretório CA3/build/test-results/ e processa-os para mostrar os resultados dos testes no Jenkins.

- Etapa Archive

```groovy
stage('Archive') {
    steps {
        // Arquivar os artefactos JAR compilados de 'CA3'
        archiveArtifacts artifacts: 'CA3/build/libs/*.jar', fingerprint: true
    }
}
```

`Archive`: Esta etapa arquiva os artefactos de compilação (neste caso, ficheiros .jar) produzidos pelo processo de compilação.

`archiveArtifacts artifacts: 'CA3/build/libs/*.jar', fingerprint: true`: Este comando arquiva os ficheiros JAR encontrados no diretório CA3/build/libs/. A opção fingerprint: true ajuda o Jenkins a rastrear estes artefactos através das compilações.

- Etapa Deploy to Production?

```groovy
stage('Deploy to Production?') {
    steps {
        input message: 'Deploy to Production?'
    }
}
```

`Deploy to Production?`: Esta etapa pausa o pipeline e solicita aprovação manual antes de prosseguir com a implementação em produção.

`input message`: O Jenkins apresenta uma mensagem ao utilizador, perguntando se está tudo bem para prosseguir com a implementação. O utilizador pode aprovar ou rejeitar a implementação.

- Etapa Deploy

```groovy
stage('Deploy') {
    steps {
        ansiblePlaybook(
            playbook: 'green.yml',
            inventory: 'hosts.ini'  
        )
    }
}
```

`Deploy`: Esta etapa implementa a aplicação num servidor usando Ansible.

`ansiblePlaybook`: Esta etapa usa o passo ansiblePlaybook fornecido pelo plugin Jenkins Ansible para executar um playbook Ansible. O playbook definido aqui é green.yml, que provavelmente contém as tarefas para implementar a aplicação num servidor remoto.

`inventory: 'hosts.ini'`: O argumento inventory diz ao Ansible quais hosts devem ser alvo para a implementação. 

- Ações Post

```groovy
post {
    success {
        echo 'Pipeline executado com sucesso!'
    }
    failure {
        echo 'Pipeline falhou!'
    }
}
```

`post`: O bloco post contém ações que são executadas após a conclusão do pipeline, com base no seu resultado (sucesso ou falha).

`success`: Se o pipeline for concluído com sucesso, irá imprimir Pipeline executado com sucesso! na consola.

`failure`: Se alguma etapa falhar, irá imprimir Pipeline falhou! na consola.

6. `rollback.yml`

O playbook `rollback.yml` é usado para reverter a aplicação em caso de falha na implementação ou problemas de produção. Para a execução do processo Java e obtém a compilação estável anterior do Jenkins para reimplementá-la.

**Secções e Funcionalidades Principais:**

- Parar aplicação: Para qualquer aplicação Java em execução usando pkill.

```yaml
- name: Parar aplicação
  shell: pkill -f java || true
```

- Obter artefacto do Jenkins: Descarrega o último artefacto .jar estável do Jenkins.

```yaml
- name: Obter artefacto do Jenkins
  uri:
    url: "http://<JENKINS_URL>/job/<JOB_NAME>/lastStableBuild/artifact/build/libs/<ARTIFACT_NAME>.jar"
    method: GET
    dest: /home/vagrant/project/app.jar
    force_basic_auth: yes
    user: "<USERNAME>"
    password: "<PASSWORD>"

```

- Implementar artefacto: Implementa o ficheiro .jar descarregado na VM green.

```yaml
- name: Implementar artefacto
  shell: java -jar /home/vagrant/project/app.jar &
```

### Resumo de Como Estes Ficheiros Funcionam em Conjunto:

**Vagrantfile**: Define e provisiona duas VMs (blue e green) usando VirtualBox e Ansible.

**hosts.ini**: Fornece as credenciais SSH necessárias para o Ansible gerir as VMs.

**blue.yml & green.yml**: Provisionam as VMs instalando as dependências necessárias e compilando a aplicação.

**Jenkinsfile**: Define o pipeline CI/CD que compila, testa e implementa a aplicação usando Jenkins e Ansible.

**rollback.yml**: Fornece um mecanismo para reverter para a versão anterior da aplicação se necessário.

Em conjunto, estes ficheiros automatizam todo o processo de compilação, teste e implementação de uma aplicação baseada em Gradle num ambiente virtualizado gerido pelo Vagrant, usando Jenkins para o pipeline CI/CD e Ansible para provisionamento e implementação.

## Pipeline CI/CD para Implementação de uma Aplicação Spring com Docker
Esta secção cobrirá os passos para criar um pipeline CI/CD para implementar uma aplicação Spring com Docker num ambiente de produção.

### Automatização da Configuração da Infraestrutura
Nos próximos passos, iremos automatizar a configuração do ambiente de produção usando Vagrant e Ansible. Este ficheiro Vagrant foi reutilizado do CA4 anterior. Para mais informações sobre como configurar Vagrant e Ansible, consulte o [README do CA4](../CA4/README.md).
#### Passo 1: Criar uma VM de Produção
Na sua pasta desejada, crie um `Vagrantfile` com a seguinte configuração:
```ruby
Vagrant.configure("2") do |config|
  # Provisionamento de chave privada para app
  config.vm.provision "file",
    source: "~/.ssh/app_key",
    destination: "/home/vagrant/.ssh/app_key"
  # Caminhos de chave privada SSH
  config.ssh.private_key_path = [
    "~/.vagrant.d/insecure_private_key",
    "~/.ssh/app_key",  # Garanta que isto existe executando `ssh-keygen -t rsa -b 2048 -f ~/.ssh/app_key`
  ]
    config.vm.define "app" do |app|
      app.vm.box = "ubuntu/bionic64"
      app.vm.network "private_network", ip: "192.168.56.20"
      app.vm.network "forwarded_port", guest: 8080, host: 8080
      app.vm.provider "virtualbox" do |vb|
        vb.memory = "2048"
        vb.cpus = 2
      end
      app.vm.provision "file",
          source: "~/.ssh/app_key.pub",
          destination: "~/.ssh/authorized_keys"
      app.vm.provision "shell", inline: <<-SHELL
        chmod 600 /home/vagrant/.ssh/app_key
      SHELL
      app.vm.provision "ansible_local" do |ansible|
        ansible.playbook = "deploy.yml"
        ansible.inventory_path = "hosts.ini"
        ansible.extra_vars = { host: "app" }
        ansible.compatibility_mode = "2.0"
        ansible.raw_arguments = ["--ssh-common-args='-o StrictHostKeyChecking=no'"]
      end
    end
    config.ssh.insert_key = false
end
```
Esta configuração criará uma VM com o endereço IP `192.168.56.20` e provisioná-la-á com o playbook Ansible `deploy.yml`.
#### Passo 2: Provisionar a VM com Ansible
- Crie um ficheiro `hosts.ini` no mesmo diretório que o `Vagrantfile` com a seguinte configuração:
```ini
[app]
192.168.56.20 ansible_user=vagrant ansible_ssh_private_key_file=/home/vagrant/.ssh/app_key
```
- Crie um ficheiro `deploy.yml` no mesmo diretório que o `Vagrantfile` com a seguinte configuração:
```yaml
---
- name: Implementar aplicação em produção
  hosts: all
  become: yes
  tasks:
    - name: Instalar Docker
      ansible.builtin.apt:
        name: docker.io
        state: present
        update_cache: yes

    - name: Garantir que o serviço Docker está em execução
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    - name: Fazer login no Docker Hub
      ansible.builtin.command: >
        docker login --username {{ docker_user }} --password {{ docker_password }}
      vars:
        docker_user: <seu_nome_de_utilizador_docker>
        docker_password: <sua_password_docker>

    - name: Obter a imagem Docker mais recente
      ansible.builtin.command: docker pull 1210902/cogsi:latest

    - name: Parar e remover o contentor antigo (se existir)
      ansible.builtin.command: >
        docker rm -f spring-app || true

    - name: Executar o novo contentor Docker
      ansible.builtin.command: >
        docker run -d --name spring-app -p 8080:8080 1210902/cogsi:latest
```
Este playbook será usado tanto no Vagrantfile como no Jenkinsfile para automatizar o processo de implementação. As tarefas são as seguintes:
1. Instalar Docker com `apt`
2. Garantir que o serviço Docker está em execução com `service`
3. Fazer login no Docker Hub com `command` e credenciais
4. Obter a imagem Docker mais recente 
5. Parar e remover o contentor antigo (se existir) 
6. Executar o novo contentor Docker 

Para provisionar a VM, execute o seguinte comando no mesmo diretório que o `Vagrantfile`:
```bash
vagrant up
```

### Configuração do Pipeline CI/CD com Jenkins
#### Executar Jenkins Localmente
- Instale o jenkins na sua máquina local a partir do [site oficial](https://www.jenkins.io/download/).
- Após instalar o Jenkins e executá-lo, navegue para `localhost:8080` no seu navegador e siga as instruções para configurar o Jenkins.
- Crie um novo item do tipo `Pipeline` e nomeie-o `example`:
  ![img.png](images/img.png)
- Vá para Configuration e configure o pipeline:
  - Na secção `Configure`, em `Build Trigger`, selecione `GitHub hook trigger for GITScm polling` para que o pipeline seja acionado automaticamente quando uma alteração for enviada para o repositório.
  - Na secção `Pipeline`, selecione `Pipeline script from SCM` e defina o `SCM` para `Git` e adicione a seguinte configuração:
    ![img_1.png](images/img_1.png)
    - Deve criar uma chave SSH e adicioná-la à sua conta GitHub para permitir que o Jenkins aceda ao repositório. Pode seguir as instruções [aqui](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh).
    - Após criar a chave SSH, adicione-a ao Jenkins navegando para `Manage Jenkins` -> `Manage Credentials` -> `Jenkins` -> `Global credentials` -> `Add Credentials`.
    ![img_2.png](images/img_2.png)
  - Em `Script Path`, defina o caminho para o `Jenkinsfile` no seu repositório.
  - Guarde a configuração.
- Crie um webhook do GitHub para acionar o pipeline quando uma alteração for enviada para o repositório:
  - No seu repositório, navegue para `Settings` -> `Webhooks` -> `Add webhook`.
  - Defina o `Payload URL` para `http://<jenkins-server>/github-webhook/`. Caso esteja a executar o Jenkins localmente, o GitHub não conseguirá aceder a ele. Pode usar uma ferramenta como [ngrok](https://ngrok.com/) para criar um túnel para o seu servidor local.
  - Defina o `Content type` para `application/json`.
  - Defina `Which events would you like to trigger this webhook?` para `Just the push event`.
  - Clique em `Add webhook`. 

Após configurar o pipeline, pode enviar uma alteração para o repositório para acionar o pipeline.

#### Definir a Lógica do Pipeline no `Jenkinsfile`
Crie um `Jenkinsfile` no diretório especificado anteriormente na configuração do Jenkins. O `Jenkinsfile` definirá a lógica do pipeline e as etapas para implementar a aplicação.
As seguintes etapas estão definidas no `Jenkinsfile`:
##### Etapa 1: Checkout
```groovy
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
```
Esta etapa faz o checkout do código do repositório, usando o comando `checkout scm`, que é um passo do pipeline Jenkins que faz checkout do código para a área de trabalho.
##### Etapa 2: Assemble
```groovy
        stage('Assemble') {
            steps {
                dir('CA2/CA2_Part2') {
                    bat './gradlew clean build --no-daemon'
                }
            }
        } 
```
Esta etapa monta a aplicação executando o comando `./gradlew clean build --no-daemon` no diretório `CA2/CA2_Part2` (diretório que contém o ficheiro `gradlew`). O comando `dir` é usado para mudar o diretório para o caminho especificado antes de executar o comando.
##### Etapa 3: Test
```groovy
    stage('Test') {
        parallel {
            stage('Testes Unitários') {
                steps {
                    dir('CA2/CA2_Part2') {
                        bat './gradlew test'
                    }
                }
            }
            stage('Testes de Integração') {
                steps {
                    dir('CA2/CA2_Part2') {
                        bat './gradlew integrationTest'
                    }
                }
            }
        }
    }
```
Esta etapa executa os testes unitários e de integração em paralelo para acelerar o processo. O comando `./gradlew test` executa os testes unitários, e o comando `./gradlew integrationTest` executa os testes de integração.
##### Etapa 4: Tag Docker Image
```groovy
        stage('Tag Docker Image') {
            steps {
                dir('CA2/CA2_Part2') {
                    bat 'docker build -t 1210902/cogsi:%BUILD_NUMBER% .'
                }
            }
        }
```
Esta etapa marca a imagem Docker com o número da compilação usando o comando `docker build -t 1210902/cogsi:%BUILD_NUMBER% .`. A variável `%BUILD_NUMBER%` é fornecida pelo Jenkins e representa o número de compilação atual.

##### Etapa 5: Archive
```groovy
        stage('Archive') {
            steps {
                dir('CA2/CA2_Part2') {
                    archiveArtifacts artifacts: '**/Dockerfile', allowEmptyArchive: true
                }
            }
        }
```
Esta etapa arquiva o `Dockerfile` no diretório `CA2/CA2_Part2` usando o comando `archiveArtifacts`. Isto permite manter um registo do `Dockerfile` usado na compilação.

##### Etapa 6: Push Docker Image
```groovy
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub') {
                        bat '''
                                docker tag 1210902/cogsi:%BUILD_NUMBER% 1210902/cogsi:%BUILD_NUMBER%
                                docker push 1210902/cogsi:%BUILD_NUMBER%
                                docker tag 1210902/cogsi:%BUILD_NUMBER% 1210902/cogsi:latest
                                docker push 1210902/cogsi:latest
                                '''
                    }
                }
            }
        }
```
Esta etapa envia a imagem Docker para o Docker Hub. O comando `docker.withRegistry` é usado para autenticar com o Docker Hub usando as credenciais fornecidas na configuração do Jenkins. O comando `docker tag` marca a imagem com o número de compilação e a tag `latest`. O comando `docker push` envia as imagens marcadas para o Docker Hub.
Certifique-se de que tem as suas credenciais Docker configuradas no Jenkins navegando para `Manage Jenkins` -> `Manage Credentials` -> `Jenkins` -> `Global credentials` -> `Add Credentials` do tipo `Username with password` com o ID `docker-hub`.
##### Etapa 7: Deploy
```groovy
stage('Deploy') {
    steps {
        when {
            branch 'main'
        }
        dir('CA6/CA6_part2') {
            bat 'ansible-playbook -i hosts.ini deploy.yml'
        }
    }
}
```
Esta etapa implementa a aplicação no ambiente de produção usando o playbook Ansible `deploy.yml` e o ficheiro de inventário `hosts.ini`. A implementação é acionada apenas quando o ramo é `main`. O comando `dir` é usado para mudar o diretório para o diretório `CA6/CA6_part2` antes de executar o comando `ansible-playbook deploy.yml`.

#### Configurar Notificações e Verificação de Implementação
##### Passo 1: Notificações
Para configurar notificações, pode usar o plugin `emailext` no Jenkins. Para instalar o plugin, navegue para `Manage Jenkins` -> `Manage Plugins` -> `Available` -> procure por `Email Extension Plugin` -> `Install without restart`.
Após instalar o plugin, pode configurar as notificações por email no script do pipeline:
```groovy
    post {
        success {
            emailext(
                subject: "Compilação Bem-sucedida: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "A compilação foi bem-sucedida.\nJob: ${env.JOB_NAME}\nCompilação: #${env.BUILD_NUMBER}",
                to: "1210902@isep.ipp.pt"
            )
        }
        failure {
            emailext(
                subject: "Compilação Falhou: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "A compilação falhou.\nJob: ${env.JOB_NAME}\nCompilação: #${env.BUILD_NUMBER}",
                to: "1210902@isep.ipp.pt"
            )
        }
    }
```
Esta configuração envia uma notificação por email para o endereço de email especificado quando a compilação é bem-sucedida ou falha.
##### Passo 2: Verificação de Implementação
- Adicione uma etapa de verificação ao pipeline na secção post para verificar a implementação:
```groovy
    post {
        success {
            script {
                def response = httpRequest url: 'http://192.168.56.20:8080/employees'
                if (response.status != 200) {
                    error("Verificação de saúde falhou com status: ${response.status}")
                } else {
                    echo "Verificação de saúde passou!"
                } 
            }
        }
    }
```
Esta etapa envia um pedido HTTP ao endpoint da aplicação implementada para verificar que a implementação foi bem-sucedida. Se o código de status não for `200`, um erro é lançado e a compilação falha. Caso contrário, uma mensagem de sucesso é impressa.

- Deve também adicionar um bloco always para fazer logout do Docker Hub após a execução do pipeline:
```groovy
    post {
        always {
            script {
                docker.withRegistry('https://index.docker.io/v1/', 'docker-hub') {
                    bat 'docker logout'
                }
            }
        }
    }
```

## Execução e Análise do Jenkinsfile
No final deve ter um `Jenkinsfile` semelhante a este:
```groovy
pipeline {
    agent any 
    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-hub')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Assemble') {
            steps {
                dir('CA2/CA2_Part2') {
                    bat './gradlew clean build --no-daemon'
                }
            }
        } 
        stage('Test') {
            parallel { 
                stage('Testes Unitários') {
                    steps {
                        dir('CA2/CA2_Part2') {
                            bat './gradlew test'
                        }
                    } 
                }
                stage('Testes de Integração') {
                    steps {
                        dir('CA2/CA2_Part2') {
                            bat './gradlew integrationTest'
                        }
                    }
                }
            }
        }
        stage('Tag Docker Image') {
            steps {
                dir('CA2/CA2_Part2') {
                    bat 'docker build -t 1210902/cogsi:%BUILD_NUMBER% .'
                }
            }
        }
        stage('Archive') {
            steps {
                dir('CA2/CA2_Part2') {
                    archiveArtifacts artifacts: '**/Dockerfile', allowEmptyArchive: true
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub') {
                        bat '''
                        docker tag 1210902/cogsi:%BUILD_NUMBER% 1210902/cogsi:%BUILD_NUMBER%
                        docker push 1210902/cogsi:%BUILD_NUMBER%
                        docker tag 1210902/cogsi:%BUILD_NUMBER% 1210902/cogsi:latest
                        docker push 1210902/cogsi:latest
                        '''
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                when {
                    branch 'main'
                }
                dir('CA6/CA6_part2') {
                    bat 'ansible-playbook -i hosts.ini deploy.yml'
                }
            }
        }
    }
    post {
        success {
            script {
                def response = httpRequest url: 'http://192.168.56.20:8080/employees'
                if (response.status != 200) {
                    error("Verificação de saúde falhou com status: ${response.status}")
                } else {
                    echo "Verificação de saúde passou!"
                }
            }
            emailext(
                subject: "Compilação Bem-sucedida: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "A compilação foi bem-sucedida.\nJob: ${env.JOB_NAME}\nCompilação: #${env.BUILD_NUMBER}",
                to: "1210902@isep.ipp.pt"
            )
        }
        always {
            script {
                docker.withRegistry('https://index.docker.io/v1/', 'docker-hub') {
                    bat 'docker logout'
                }
            }
        }
        failure {
            emailext(
                subject: "Compilação Falhou: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "A compilação falhou.\nJob: ${env.JOB_NAME}\nCompilação: #${env.BUILD_NUMBER}",
                to: "1210902@isep.ipp.pt"
            )
        }
    }
}
```

Após configurar o pipeline, pode enviar uma alteração para o repositório para acionar o pipeline. O pipeline irá automaticamente compilar a aplicação, executar testes, compilar e enviar a imagem Docker, implementar a aplicação no ambiente de produção e enviar notificações por email.
- Aqui está como a execução do seu pipeline deve parecer:
![img_4.png](images/img_4.png)

- Também pode encontrar os logs de cada etapa na saída da consola do Pipeline:
![img_5.png](images/img_5.png)

- O arquivo do Dockerfile na secção `Artifacts`:
![img_6.png](images/img_6.png)

- Finalmente, aceda ao endereço IP da sua máquina na porta 8080 e deverá ver a aplicação em execução:
![img_3.png](images/img_3.png)

### Problemas Comuns
- Se encontrar problemas com os comandos Docker no pipeline, certifique-se de que o Docker está instalado no servidor Jenkins e que o utilizador Jenkins tem as permissões necessárias para executar comandos Docker.
- Se encontrar problemas com o playbook Ansible, certifique-se de que o Ansible está instalado no servidor Jenkins e que o playbook está corretamente configurado.
- Se encontrar problemas com as notificações por email, certifique-se de que o `Email Extension Plugin` está instalado no Jenkins e que a configuração de email está correta.
- Se encontrar problemas com a etapa de implementação, certifique-se de que o playbook está corretamente configurado e que o servidor Jenkins pode aceder à VM de produção.
- Certifique-se de que as suas credenciais estão corretamente configuradas no Jenkins.
- Verifique se tem todos os seguintes plugins instalados no Jenkins:
  - Pipeline
  - Pipeline: GitHub
  - Email Extension Plugin
  - HTTP Request Plugin
  - Docker Pipeline
  - Docker Commons
  - Docker
  - Ansible

## Análise

### Jenkins vs. GitHub Actions

*   **Jenkins**: É um servidor de automação *open-source* criado para suportar a criação, teste e *deployment* de software. Funciona como uma solução *self-hosted*, onde o utilizador é responsável por gerir o servidor, o sistema operativo e as dependências. É conhecido pela sua extrema flexibilidade e por um vasto sistema de plugins, permitindo a integração com praticamente qualquer ferramenta através de scripts  (geralmente em Groovy).

*   **GitHub Actions**: É uma plataforma de CI/CD (Integração Contínua e Entrega Contínua) baseada na cloud e  integrada nos repositórios GitHub. Ao contrário do Jenkins, opera num modelo SaaS (*Software as a Service*), eliminando a necessidade de gestão de servidores para a maioria dos casos de uso. Utiliza ficheiros YAML para definir *workflows* automatizados e tira partido de um ecossistema de "Actions" pré-construídas pela comunidade e verificadas pelo GitHub, facilitando a configuração rápida de *pipelines*.

| **Aspeto** | **Jenkins**                                                                                                                                                                          | **GitHub Actions**                                                                                                                                         |
| :--- |:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Infraestrutura e Manutenção** | Modelo *Self-Hosted*. Requer instalação, configuração manual do servidor, atualizações de segurança e gestão de capacidade dos agentes de *build*.                                   | Modelo *Cloud-native*. A infraestrutura é gerida pelo GitHub geralmente mas suporta *self-hosted runners* se necessário.                                   |
| **Integração com Controlo de Versão** | Externa. Liga-se ao GitHub (ou outros git remotos) via *webhooks* e plugins.                                                                                                         | Nativa. Os *checks* de CI/CD aparecem diretamente na interface dos *Pull Requests* e a configuração reside no próprio repositório (`.github/workflows`).   |
| **Configuração de Pipelines** | Baseada em *Jenkinfiles* (declarativos ou *scripted*), frequentemente utilizando a linguagem Groovy.      | Baseada em ficheiros YAML. Sintaxe mais simples e legível focada na composição de tarefas.                                                                 |
| **Escalabilidade** | Manual ou requer orquestração complexa. É necessário configurar nós adicionais (*slaves*) e gerir o balanceamento de carga.                                                          | Automática no modelo *cloud*. O GitHub provisiona ambientes de execução (*runners*) conforme a necessidade dos *jobs* paralelos.                           |
| **Segurança** | Da responsabilidade do administrador. É necessário garantir o *hardening* do servidor Jenkins e gerir permissões de utilizadores granularmente.                                      | Gerida pela plataforma. Oferece gestão de segredos encriptados e conformidade de segurança gerida pelo GitHub.                                             |

### Principais diferenças técnicas

#### 1. Arquitetura de Execução (Stateful vs. Stateless)
A diferença fundamental reside na persistência do ambiente.
*   **No Jenkins:** O servidor é persistente (*stateful*). O ambiente de *build* pode acumular "lixo" (ficheiros temporários, configurações globais alteradas) se não for limpo após cada execução.
*   **No GitHub Actions:** Cada *job* é executado num ambiente virtual (VM ou Container) *stateless*. Isto garante que o ambiente de teste é sempre limpo e idêntico.
#### 2. Ecossistema de Extensões
*   **Plugins do Jenkins:** São instalados ao nível do servidor. Uma atualização de um plugin para um projeto pode quebrar a *pipeline* de outro projeto que partilhe o mesmo servidor.
*   **Marketplace do GitHub:** As *Actions* são referenciadas por versão no código do *workflow* (ex: `actions/checkout@v3`). Isto permite que diferentes projetos usem diferentes versões da mesma ferramenta sem conflitos.

#### 3. Barreira de Entrada
Para equipas que já utilizam GitHub, a fricção inicial do Actions é praticamente nula.
*   **Contexto:** O GitHub Actions tem acesso imediato ao contexto do repositório (código, *issues*, *tags*) sem configuração adicional. O Jenkins necessita da configuração de credenciais, chaves SSH e *webhooks* bidirecionais para atingir o mesmo nível de automação.

## IMPLEMENTAÇÃO

#### Configuração dos Self-Hosted Runners

Para esta implementação, optámos por utilizar self-hosted runners para executar os nossos workflows do GitHub Actions, em vez dos runners do GitHub.

Esta abordagem foi necessária para garantir que a pipeline  tivesse acesso direto à nossa infraestrutura (máquinas virtuais criadas no VirtualBox). Configurámos o runner na nossa máquina  seguindo os passos fornecidos pelo GitHub em Settings > Actions > Runners > New self-hosted runner.

#### Part 1: CI/CD Pipeline for Deploying Building REST services with Spring application and deploys it to a local virtual machine using GitHub Actions
- Criar um ficheiro chamado .github/workflows/ci-dc.yml na base do repositório com o seguinte conteúdo:
```yaml
name: CI/CD Pipeline

permissions:
  contents: write

on:
  push:
    branches: [ main, development ]
  workflow_dispatch:

env:
  ARTIFACT_NAME: rest-service-0.0.1-SNAPSHOT.jar
  APP_PATH: CA2-part2/tut-gradle

jobs:
  build-and-test:
    runs-on: self-hosted

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Make gradlew executable
        run: chmod +x ./gradlew
        working-directory: ${{ github.workspace }}/CA2-part2/tut-gradle

      - name: Assemble
        run: |
          echo "Assembling the application..."
          ./gradlew clean assemble
        working-directory: ${{ github.workspace }}/CA2-part2/tut-gradle

      - name: Test
        run: |
          echo "Running unit tests..."
          ./gradlew test
        working-directory: ${{ github.workspace }}/CA2-part2/tut-gradle

      - name: Publish Test Results
        if: always() && (github.event_name == 'push' || github.event_name == 'workflow_dispatch')
        run: |
          REPORT_PATH="${{ github.workspace }}/CA2-part2/tut-gradle/build/test-results/test/*.xml"
          shopt -s nullglob
          files=($REPORT_PATH)
          if [ ${#files[@]} -gt 0 ]; then
            echo "Publishing test reports..."
            echo "path=$REPORT_PATH" >> $GITHUB_OUTPUT
          else
            echo "No test reports found, skipping."
          fi
        shell: bash

      - name: Run Test Reporter
        uses: dorny/test-reporter@v1
        if: steps.publish-reports.outputs.path != ''
        with:
          name: Test Results
          path: ${{ steps.publish-reports.outputs.path }}
          reporter: java-junit

      - name: Tag stable build
        if: success()
        run: |
          echo "Tagging stable build..."
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git tag stable-v${{ github.run_number }}
          git push origin stable-v${{ github.run_number }}

      - name: Archive Artifact
        uses: actions/upload-artifact@v4
        if: success()
        with:
          name: application-jar
          path: ${{ github.workspace }}/CA2-part2/tut-gradle/build/libs/*.jar

  deploy-approval:
    needs: build-and-test
    runs-on: self-hosted
    if: success()
    environment:
      name: production
      url: http://192.168.56.12:8080

  deploy:
    needs: deploy-approval
    runs-on: self-hosted
    if: success()

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Download Artifact
        uses: actions/download-artifact@v4
        with:
          name: application-jar
          path: ./artifacts

      - name: Set up Python for Ansible
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Install Ansible
        run: |
          pip install ansible

      - name: Check VM is Ready
        run: |
          echo "Checking if green VM is accessible..."
          until ssh -o StrictHostKeyChecking=no vagrant@192.168.56.12 'echo ready' 2>/dev/null; do
            echo "Waiting for VM..."
            sleep 5
          done
          echo "VM is ready!"


      - name: Deploy to Green VM
        env:
          ARTIFACT_PATH: ${{ github.workspace }}/artifacts/${{ env.ARTIFACT_NAME }}
        run: |
          echo "Deploying to Green VM..."
          ansible-playbook -i CA6/part1/host.ini CA6/part1/green.yml \
            --extra-vars "artifact_path=${ARTIFACT_PATH}"

      - name: Health Check
        if: success()
        run: |
          echo "Running health check on Green VM..."
          for i in {1..5}; do
            if curl -f http://192.168.56.12:8080/employees; then
              echo "Health check passed!"
              exit 0
            fi
            echo "Attempt $i failed, retrying..."
            sleep 5
          done
          echo "Health check failed!"
          exit 1

```

Este ficheiro replica as fases e funcionalidades do nosso Jenkinsfile.
1. Build and Test
- O job build-and-test:
    - Faz o checkout do código..
    - Configura o JDK 17.
    - Constrói a aplicação e executa os testes.
    - Carrega os resultados dos testes e os artefactos de construção (ficheiros JAR) como artefactos para serem usados.
2. Deployment to Green VM
- O job deploy-to-green:
    - Depende do sucesso de build-and-test.
    - É acionado apenas quando iniciado manualmente tendo que ser aprovado no environment das actions.
    - Instala o Ansible e executa o playbook green.yml para dar deploy à aplicação na VM green.



#### Part 2: CI/CD Pipeline for Deploying a Spring Application with Docker using GitHub Actions
- Criar um ficheiro deploy-spring.yml no diretório .github/workflows com a seguinte configuração

```yaml
   name: Part 2 - Docker Deployment Pipeline

   permissions:
     contents: write

   on:
     push:
       branches: [ main, development ]
     workflow_dispatch:

   env:
     DOCKER_IMAGE: 1210902/spring-rest-service
     APP_PATH: CA2-part2/tut-gradle

   jobs:
     build-and-test:
       runs-on: self-hosted

       steps:
         - name: Checkout
           uses: actions/checkout@v4
           with:
             fetch-depth: 0

         - name: Set up JDK 17
           uses: actions/setup-java@v4
           with:
             java-version: '17'
             distribution: 'temurin'

         - name: Make gradlew executable
           run: chmod +x ./gradlew
           working-directory: ${{ github.workspace }}/CA2-part2/tut-gradle

         - name: Assemble
           run: |
             echo "Assembling the application..."
             ./gradlew clean assemble
           working-directory: ${{ github.workspace }}/CA2-part2/tut-gradle

         - name: Test
           run: |
             echo "Running unit tests..."
             ./gradlew test
           working-directory: ${{ github.workspace }}/CA2-part2/tut-gradle

         - name: Publish Test Results
           if: always()
           uses: dorny/test-reporter@v1
           with:
             name: Test Results
             path: ${{ github.workspace }}/CA2-part2/tut-gradle/build/test-results/test/*.xml
             reporter: java-junit

         - name: Set Docker Tag
           id: docker_tag
           run: |
             if [ "${{ github.ref }}" == "refs/heads/main" ]; then
               echo "tag=latest" >> $GITHUB_OUTPUT
             else
               echo "tag=${{ github.ref_name }}-${{ github.run_number }}" >> $GITHUB_OUTPUT
             fi

         - name: Build Docker Image
           run: |
             echo "Building Docker image: ${{ env.DOCKER_IMAGE }}:${{ steps.docker_tag.outputs.tag }}"
             docker build -t ${{ env.DOCKER_IMAGE }}:${{ steps.docker_tag.outputs.tag }} .
           working-directory: ${{ github.workspace }}/CA2-part2/tut-gradle

         - name: Login to Docker Hub
           uses: docker/login-action@v3
           with:
             username: ${{ secrets.DOCKER_USERNAME }}
             password: ${{ secrets.DOCKER_PASSWORD }}

         - name: Push Docker Image
           run: |
             echo "Pushing Docker image to Docker Hub..."
             docker push ${{ env.DOCKER_IMAGE }}:${{ steps.docker_tag.outputs.tag }}

             # Also push as 'latest' if on main branch
             if [ "${{ github.ref }}" == "refs/heads/main" ]; then
               docker tag ${{ env.DOCKER_IMAGE }}:${{ steps.docker_tag.outputs.tag }} ${{ env.DOCKER_IMAGE }}:latest
               docker push ${{ env.DOCKER_IMAGE }}:latest
             fi

         - name: Tag stable build
           if: success() && github.event_name == 'push'
           run: |
             echo "Tagging stable build..."
             git config user.name "github-actions"
             git config user.email "github-actions@github.com"
             git tag stable-part2-v${{ github.run_number }}
             git push origin stable-part2-v${{ github.run_number }}

     deploy-approval:
       needs: build-and-test
       runs-on: self-hosted
       if: success() && github.ref == 'refs/heads/main'
       environment:
         name: production-docker
         url: http://192.168.56.20:8080

       steps:
         - name: Manual Approval
           run: |
             echo "Manual approval granted. Proceeding to deploy..."

     deploy:
       needs: deploy-approval
       runs-on: self-hosted
       if: success()

       steps:
         - name: Checkout
           uses: actions/checkout@v4

         - name: Set up Python for Ansible
           uses: actions/setup-python@v5
           with:
             python-version: '3.x'

         - name: Install Ansible
           run: |
             pip install ansible

         - name: Check Production VM is Ready
           run: |
             echo "Checking if Production VM is accessible..."
             until ssh -o StrictHostKeyChecking=no vagrant@192.168.56.20 'echo ready' 2>/dev/null; do
               echo "Waiting for VM..."
               sleep 5
             done
             echo "VM is ready!"

         - name: Deploy to Production VM
           env:
             DOCKER_IMAGE: ${{ env.DOCKER_IMAGE }}:latest
             DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
             DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
           run: |
             echo "Deploying Docker container to Production VM..."
             ansible-playbook -i CA6/part2/host.ini CA6/part2/deploy.yml \
               --extra-vars "docker_image=${DOCKER_IMAGE} docker_username=${DOCKER_USERNAME} docker_password=${DOCKER_PASSWORD}"

         - name: Health Check
           if: success()
           run: |
             echo "Running health check on Production VM..."
             for i in {1..10}; do
               if curl -f http://192.168.56.20:8080/employees; then
                 echo "✓ Health check passed!"
                 exit 0
               fi
               echo "Attempt $i failed, retrying..."
               sleep 5
             done
             echo "✗ Health check failed!"
             exit 1
```
Esta configuração define um workflow do GitHub Actions que automatiza a pipeline de CI/CD para dar deploy a uma aplicação Spring com Docker.
O workflow é acionado em eventos de push para a branch main e executa os seguintes passos.

1. Instalar as dependências necessárias.
2. Fazer o checkout do código do repositório.
3. Montar o projeto executando o comando ./gradlew clean assemble.
4. Executar testes unitários e de integração.
5. Construir e etiquetar a imagem Docker.
6. Enviar a imagem Docker para o Docker Hub.
7. Implantar a aplicação usando Ansible.
8. Realizar uma verificação de integridade (health check) na aplicação implantada.

- Após criar o ficheiro do workflow, fazemos o push para o repositório para acionar a pipeline.
- Podemos aceder ao separador GitHub Actions no repositório para ver as execuções do workflow e os logs.