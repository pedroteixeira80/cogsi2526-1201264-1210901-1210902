# CA3 - Virtualização com Vagrant
## Self-evaluation
Hélder Rocha (1210901) - 33,33%
Pedro Teixeira (1210902) - 33,33%
Francisco Gouveia(1201264) - 33,33%

## Parte 1 - Configuração do Vagrant para Implementação de VM

### Visão Geral
Este projeto utiliza o Vagrant para configurar um ambiente virtual para executar um serviço REST Spring Boot com uma base de dados H2. A configuração está dividida em duas partes para simular o alojamento da aplicação e da base de dados em servidores separados.

### Passos
1. **Instalar o Vagrant**

Para [instalar](https://developer.hashicorp.com/vagrant/install) o Vagrant. Escolha o sistema operativo e arquitetura corretos para o seu computador principal

2. **Instalar o plugin do Vagrant para adições de convidado**
```bash
vagrant plugin install vagrant-vbguest
```

3. **Criar um novo diretório para o projeto**
```bash
mkdir vagrant_demo
cd vagrant_demo
``` 

4. **Inicializar o Vagrant**
```bash
vagrant init
```
- Isto irá criar um ficheiro chamado Vagrantfile.

5. **Escolher uma box base**
- Abra o Vagrantfile num editor de texto e faça as seguintes modificações:
- config.vm.box: Define a box base, que é a imagem do sistema operativo para a VM—neste caso, Ubuntu 18.04 (bionic64).
```groovy
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
```

6. **Configurar a rede**
```groovy
config.vm.network "private_network", ip: "192.168.56.10"
config.vm.network "forwarded_port", guest: 8080, host: 8080
```
- Configura a VM com um endereço IP estático na rede privada. 
- Isto permite um acesso mais fácil à VM a partir do computador principal utilizando o IP especificado.
- Configura o redirecionamento de portas, expondo a porta 8080 na VM para a porta 8080 no computador principal, permitindo que aplicações (como um servidor Spring Boot) sejam acedidas a partir do computador principal através de localhost:8080.

7. **Pasta Sincronizada para Armazenamento da Base de Dados H2**
```groovy
config.vm.synced_folder "./h2_data", "/home/vagrant/h2_data"
```
- Sincroniza uma pasta local (./h2_data) com uma pasta dentro da VM (/home/vagrant/h2_data). Isto é útil para armazenamento persistente de dados, como dados da base de dados H2, permitindo que sejam acessíveis tanto do computador principal como da VM.

8. **Variáveis de Ambiente e Provisionamento Básico**
- Estas variáveis de ambiente são configuradas externamente e passadas ao Vagrant. Controlam algumas ações no script de provisionamento:
 - CLONE_REPO: Se definido como true, o script irá clonar o repositório Git.
 - BUILD_RUN_GRADLE_APP e BUILD_RUN_MAVEN_APP: Quando definidos como true, estes controlam a compilação e execução dos projetos Gradle e Maven, respetivamente.
 - GITHUB_TOKEN: Token de autenticação do GitHub para clonar um repositório privado.
```groovy
CLONE_REPO = ENV['CLONE_REPO']
BUILD_RUN_GRADLE_APP = ENV['BUILD_RUN_GRADLE_APP']
BUILD_RUN_MAVEN_APP = ENV['BUILD_RUN_MAVEN_APP']
GITHUB_TOKEN = ENV['GITHUB_TOKEN']
```

9. **Passar variáveis de ambiente para o script de provisionamento**
```groovy
config.vm.provision "shell", inline: <<-SHELL
    echo "CLONE_REPO=#{CLONE_REPO}"
    echo "BUILD_RUN_GRADLE_APP=#{BUILD_RUN_GRADLE_APP}"
    echo "BUILD_RUN_MAVEN_APP=#{BUILD_RUN_MAVEN_APP}"
```
Script de Provisionamento:

10. **Instalação Básica de Pacotes e Configuração**
Esta secção instala pacotes essenciais:
- Git para controlo de versões.
- JDK (Java Development Kit) versão 17, necessário para aplicações Java.
- Maven e Gradle, duas ferramentas de compilação usadas para projetos Java. O Maven é instalado através do apt-get, enquanto o Gradle é descarregado e instalado manualmente porque é especificada uma versão mais recente do que a fornecida pelo apt-get.
```bash
# Atualizar lista de pacotes
sudo apt-get update

# Instalar Git
sudo apt-get install -y git

# JDK (OpenJDK 17)
sudo apt-get install -y openjdk-17-jdk

# Maven
sudo apt-get install -y maven

# Gradle
sudo apt-get install -y wget unzip
wget https://services.gradle.org/distributions/gradle-7.6-bin.zip -P /tmp
sudo unzip -d /opt/gradle /tmp/gradle-7.6-bin.zip
sudo ln -s /opt/gradle/gradle-7.6/bin/gradle /usr/bin/gradle
```

11. **Verificação de Versão**
Após instalar cada ferramenta, esta secção verifica se as instalações foram bem-sucedidas imprimindo as suas versões.
```bash
git --version
java -version
mvn -version
gradle -v
```

12. **Clonagem do Repositório**
Esta parte do script clona um repositório GitHub na VM.
- Verifica se a variável de ambiente CLONE_REPO está definida como "true".
- Se for true, utiliza o token do GitHub armazenado em GITHUB_TOKEN para autenticar e clonar o repositório.
- Se for false, ignora este passo.
Esta configuração torna o passo de clonagem flexível, permitindo que seja ativado ou desativado através da definição da variável de ambiente.
```bash
if [[ "#{CLONE_REPO}" == "true" ]]; then
    echo "A clonar o repositório..."
    git clone https://#{GITHUB_TOKEN}@github.com/pedroteixeira80/cogsi2526-1201264-1210901-1210902.git /home/vagrant/project
else
    echo "A ignorar a clonagem do repositório..."
fi
```

13. **Alterar o Diretório de Trabalho**
É usado para alterar o diretório de trabalho dentro da máquina virtual para /home/vagrant/project.
Este é o diretório onde o repositório do projeto é clonado quando CLONE_REPO está definido como "true".
```bash
cd /home/vagrant/project
```

14. **Compilar e Executar a Aplicação Gradle**
Esta secção compila um projeto Gradle se a variável de ambiente BUILD_RUN_GRADLE_APP estiver definida como "true".
- Muda para o diretório CA2, onde o projeto Gradle está localizado.
- Executa ./gradlew build, um comando wrapper do Gradle que compila o projeto (por exemplo, compila o código e cria os ficheiros necessários).
- Se BUILD_RUN_GRADLE_APP não for "true", ignora esta parte.
```bash
if [[ "#{BUILD_RUN_GRADLE_APP}" == "true" ]]; then
    echo "A compilar a aplicação gradle..."
    cd CA2
    ./gradlew build
    cd ..
else
    echo "A ignorar a compilação da aplicação gradle..."
fi
```

15. **Compilar e Executar a Aplicação Maven**
Esta parte trata da compilação e execução da aplicação Maven se BUILD_RUN_MAVEN_APP estiver definido como "true".
- Navega para o diretório CA1/nonrest, onde o projeto Maven reside.
- Executa ../mvnw spring-boot:run, que utiliza o wrapper do Maven (mvnw) para executar a aplicação Spring Boot.
- Se BUILD_RUN_MAVEN_APP não for "true", este passo é ignorado.
```bash
if [[ "#{BUILD_RUN_MAVEN_APP}" == "true" ]]; then
    echo "A iniciar a aplicação maven..."
    cd CA1/nonrest
    ../mvnw spring-boot:run
    cd ..
else
    echo "A ignorar o início da aplicação maven..."
fi
```

### Comandos essenciais
Este documento descreve os comandos essenciais do Vagrant para gerir e interagir com a sua máquina virtual (VM) conforme definido no seu `Vagrantfile`.

#### `vagrant up`
Execute vagrant up quando quiser iniciar ou criar a máquina virtual definida no seu Vagrantfile.
Este comando irá configurar a VM de acordo com a configuração no Vagrantfile, instalando a box especificada (como Ubuntu), configurando as configurações de rede e executando scripts de provisionamento.
```bash
vagrant up
```
Ao executar este comando, o Vagrant:
Descarrega e inicializa a box base se ainda não estiver no seu sistema.
Aplica configurações como configuração de rede, pastas partilhadas e limites de recursos (CPU, memória).
Executa quaisquer scripts de provisionamento, como provision.sh, para instalar dependências ou ferramentas na VM.

#### `vagrant ssh`
Use vagrant ssh após vagrant up para aceder à interface de terminal da VM. 
Este comando permite-lhe entrar na máquina virtual, permitindo-lhe trabalhar diretamente dentro da VM.
```bash
vagrant ssh
```
O comando vagrant ssh fornece-lhe acesso direto à VM, permitindo-lhe:
Executar comandos, depurar problemas e instalar ou configurar manualmente ferramentas adicionais.
Verificar se as instalações dos seus scripts de provisionamento foram bem-sucedidas.
Interagir diretamente com os seus projetos e executar aplicações dentro da VM como se estivesse a trabalhar numa máquina separada.

#### `vagrant provision`
Se alterou apenas o conteúdo de provision.sh, pode simplesmente reaprovisionar a máquina sem precisar de a recriar:
```bash
vagrant provision
```
Este comando executa novamente o script de provisionamento, aplicando as alterações feitas à configuração e instalações dentro da máquina virtual sem precisar de recriar a VM.

#### `vagrant reload --provision`
Se modificou o Vagrantfile (por exemplo, alterando configurações de rede, sincronização de pastas ou recursos da VM), é recomendado recarregar a máquina virtual:
```bash
vagrant reload --provision
```
O comando reload reinicia a máquina virtual, e com a opção --provision, também executa novamente o script de provisionamento após o reinício, aplicando as novas configurações tanto do Vagrantfile como do provision.sh (se ambos foram modificados).


## Parte 2 - Utilização de diferentes Servidores

### Visão Geral
O objetivo da Parte 2 desta tarefa é usar o Vagrant para configurar
um ambiente virtual com duas VMs para executar a versão Gradle
da aplicação Building REST services with Spring. Uma VM deve alojar a aplicação Spring (app), enquanto a
outra deve alojar a base de dados H2 (db). Esta configuração permitir-lhe-á simular um cenário do mundo real onde a
aplicação e a base de dados são alojadas em servidores separados, facilitando uma melhor compreensão e gestão da comunicação entre serviços.

### Passos
1. **Criar as duas VMs**
- Tal como na parte 1, execute `vagrant init` na pasta desejada.
- Modifique o Vagrantfile com uma configuração como esta:
```groovy
    Vagrant.configure("2") do |config|
      # Definir a VM da base de dados
      config.vm.define "db" do |db|
        db.vm.box = "ubuntu/bionic64" # Definir o tipo de VM
        db.vm.network "private_network", ip: "192.168.56.11" # Definir o IP da VM, isto é útil para identificar as máquinas
        db.vm.provider "virtualbox" do |vb|
          vb.memory = "1024" # Definir a quantidade de memória
          vb.cpus = 1 # Definir a quantidade de CPUs
        end
      end
    
      # Definir a VM da aplicação
      config.vm.define "app" do |app|
        app.vm.box = "ubuntu/bionic64" # Definir o tipo de VM 
        app.vm.network "private_network", ip: "192.168.56.12" # Definir o IP da VM, isto é útil para identificar as máquinas
        app.vm.network "forwarded_port", guest: 8080, host: 8080 # Definir o redirecionamento de portas
        app.vm.provider "virtualbox" do |vb|
          vb.memory = "2048" # Definir a quantidade de memória
          vb.cpus = 2 # Definir a quantidade de CPUs
        end
      end
    end
```
- Execute `vagrant up` para criar as VMs.
- Adicione scripts de provisionamento para ajudar na clonagem, compilação e início da aplicação.
  - O seguinte ficheiro é um exemplo de um script que automatiza o processo de clonagem, compilação e início de uma aplicação Spring-Boot.
```bash
    #!/bin/bash
    # Instalar Java e Git
    sudo apt-get update
    sudo apt-get install -y openjdk-17-jdk
    sudo apt-get install -y git
    # Clonar o repositório da sua aplicação Spring Boot
    git clone https://${GITHUB_TOKEN}@github.com/pedroteixeira80/cogsi2526-1201264-1210901-1210902.git /home/vagrant/app
    git config --global user.email "o.seu.email"
    git config --global user.name "o.seu.nome"
    # Compilar e iniciar a aplicação Spring Boot
    chmod -R +x /home/vagrant/app
    cd /home/vagrant/app/CA1/nonrest #Modifique para a pasta da sua app
    ../mvnw spring-boot:run
```
- Adicione um ficheiro de provisionamento à VM db para instalar Java e a base de dados H2, bem como iniciar uma base de dados H2 em modo Servidor.
```bash
  #!/bin/bash
  # Instalar Java e pacotes necessários
  sudo apt-get update
  sudo apt-get install -y openjdk-17-jdk
  sudo apt-get install -y unzip
  # Definir o diretório e ficheiros H2
  H2_DIR="/opt/h2"
  H2_ZIP="h2-2019-10-14.zip"
  H2_JAR="$H2_DIR/h2/bin/h2*.jar"
  # Verificar se o diretório H2 já existe ou se a base de dados já está a correr
  if [[ ! -d "$H2_DIR" ]]; then
  echo "Diretório H2 não encontrado. A descarregar base de dados H2..."
  # Descarregar o ficheiro zip da base de dados H2
  wget http://www.h2database.com/$H2_ZIP
  # Criar o diretório para H2
  mkdir -p $H2_DIR
  # Descompactar o ficheiro descarregado no diretório /opt/h2
  unzip -o $H2_ZIP -d $H2_DIR
  echo "Base de dados H2 descarregada e extraída."
  else
  echo "Diretório H2 já existe. A ignorar descarga."
  fi
  # Verificar se H2 já está a correr (verificando se a porta está em uso)
  if ! netstat -tuln | grep -q ":9092"; then
  echo "A iniciar servidor H2..."
  # Iniciar o servidor H2 usando o caminho correto para h2.jar
  java -cp $H2_JAR org.h2.tools.Server -tcp -tcpAllowOthers -tcpPort 9092 -web -webAllowOthers -webPort 8082 -ifNotExists &
  else
  echo "Servidor H2 já está a correr na porta 9092."
  fi
```
  - Adicione a seguinte configuração ao seu VagrantFile para executar os scripts.
```groovy 
    db.vm.provision "shell", path: "db.sh" #Adicione esta linha à configuração db
    app.vm.provision "shell", path: "app.sh" #Adicione esta linha à configuração app
```
- Execute `vagrant up --provision` para aprovisionar as VMs, faça isto sempre que quiser que as suas alterações sejam carregadas na VM.
- Adicione a seguinte configuração ao seu ficheiro app.sh para se conectar à base de dados H2.
```bash
    #!/bin/bash
    mkdir -p /home/vagrant/app/CA1/nonrest/src/main/resources
    echo "
    spring.datasource.url=jdbc:h2:tcp://192.168.56.11:9092/~/test;DB_CLOSE_ON_EXIT=FALSE 
    spring.datasource.driverClassName=org.h2.Driver
    spring.datasource.username=sa
    spring.datasource.password=
    spring.h2.console.enabled=true
    spring.jpa.hibernate.ddl-auto=update
    " > /home/vagrant/app/CA1/nonrest/src/main/resources/application.properties
```
    Isto criará um ficheiro application.properties na pasta resources da sua aplicação Spring Boot, e configurará a aplicação para se conectar à base de dados H2 a correr na VM db.
- Execute `vagrant up --provision` para aprovisionar as VMs.
- Aceda à aplicação indo a `http://192.168.56.12/employees" no seu navegador.
- Aceda à base de dados indo a `http://192.168.56.11/h2-console" no seu navegador.
- Automatize o processo de arranque para que a aplicação Spring Boot aguarde que a base de dados H2 esteja pronta adicionando o seguinte código ao seu app.sh
```bash
    #!/bin/bash
    # Aguardar que a base de dados H2 esteja pronta
    until nc -z -v -w30 192.168.56.11 9092
    do
    echo "A aguardar o início da VM db..."
    sleep 5
    done
```
- Proteja a VM db adicionando regras de firewall para restringir o acesso apenas à VM app. Adicione este código ao db.sh.
```bash
    # Instalar UFW (Uncomplicated Firewall)
    sudo apt-get install -y ufw
    # Definir políticas predefinidas
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    # Permitir SSH (opcional, para acesso remoto)
    sudo ufw allow ssh
    # Permitir ligações à porta da base de dados H2 (9092) a partir da VM app (192.168.56.12)
    sudo ufw allow from 192.168.56.12 to any port 9092
    # Ativar UFW
    yes | sudo ufw enable
```
- Melhore a segurança das suas VMs e previna acesso não autorizado usando chaves SSH personalizadas para um acesso seguro.
  - Execute os seguintes comandos para criar as chaves:
    - `ssh-keygen -t rsa -b 2048 -f ~/.ssh/app_key`
    - `ssh-keygen -t rsa -b 2048 -f ~/.ssh/db_key`
  - Adicione a seguinte configuração ao seu VagrantFile para usar as chaves.
```groovy
    # Provisionamento de chaves públicas para SSH
    config.vm.provision "file",
    source: "~/.ssh/app_key.pub",
    destination: "~/.ssh/authorized_keys"
    config.vm.provision "file",
    source: "~/.ssh/db_key.pub",
    destination: "~/.ssh/authorized_keys"
    # Caminhos das chaves privadas SSH
    config.ssh.private_key_path = [
     "~/.vagrant.d/insecure_private_key",
     "~/.ssh/app_key",  
     "~/.ssh/db_key"   
    ]
    config.ssh.insert_key = false
```
  Isto copiará as chaves públicas para o ficheiro authorized_keys nas VMs, e usará as chaves privadas para acesso SSH.
- Aceda às VMs usando os seguintes comandos:
  - `ssh -i ~/.ssh/app_key vagrant@192.168.56.12`
  - `ssh -i ~/.ssh/db_key vagrant@192.168.56.11`
  
- Os seus ficheiros finais devem ser semelhantes aos deste repositório.
  - VagrantFile: pasta CA3/part2/VagrantFile
  - app.sh: pasta CA3/part2/provision_app.sh
  - db.sh: pasta CA3/part2/provision_db.sh

# Alternative Solution Analysis: Docker vs. Vagrant

## 1. Alternative Technological Solution

An alternative to the VM-based approach of Vagrant is **containerization**, specifically using **Docker** and **Docker Compose**.  
This changes from virtualizing an entire hardware stack to virtualizing only the application's environment. Docker packages an application and its dependencies into lightweight, portable units called **containers**, while Docker Compose orchestrates multi-container applications.

| Feature / Aspect       | Vagrant (with VirtualBox) | Docker Compose |
| :--------------------- | :------------------------- | :-------------- |
| **Virtualization Type** | **Full Virtualization (VMs)**. Uses a hypervisor to run an independent guest operating system with its own kernel. | **OS-Level Virtualization (Containers)**. Uses containers for application-level virtualization, sharing the host OS's kernel. |
| **Isolation** | **Complete OS-level Isolation.** Each VM is an independent system offering strong separation from the host and other VMs. | **Application-level Isolation.** Containers are isolated from each other but share the host kernel. |
| **Startup Speed** | **Slow (Minutes).** Startup is slow, as each VM must go through the entire operating system boot process. | **Fast (Seconds).** Offers almost instant startup, as containers use the host's kernel and do not need to boot an operating system. |

---

## 2. Implementation with Docker and Docker Compose

This section details how Docker and Docker Compose were used to fulfill the requirements.

---

### Part 1: All-in-One Environment in a Single Container

The goal of Part 1 was to create a single, self-contained environment that clones, builds, and runs both the Spring Boot REST service and the Gradle chat application.

#### Dockerfile for the Unified Environment

This Dockerfile is responsible for creating a base image with all the necessary tools and dependencies installed.

- **Base Image:** 'ubuntu:22.04'
- **Dependencies:** Installs 'git', 'openjdk-17-jdk', 'maven', and a specific version of 'gradle' to handle different build systems.
- **Exposed Ports:** 8080 for the Spring web app, and 59001 for the chat server.

```Dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y     git     openjdk-17-jdk     maven     wget     unzip     && rm -rf /var/lib/apt/lists/*

# Install Gradle
RUN wget https://services.gradle.org/distributions/gradle-8.5-bin.zip -P /tmp     && unzip -d /opt/gradle /tmp/gradle-8.5-bin.zip     && rm /tmp/gradle-8.5-bin.zip

ENV GRADLE_HOME=/opt/gradle/gradle-8.5
ENV PATH=${GRADLE_HOME}/bin:${PATH}

# Set working directory
WORKDIR /app

# Expose ports
EXPOSE 8080 59001

# Default command
CMD ["/bin/bash"]
```

#### Automating Cloning, Building, and Starting Applications

- **'docker-compose.yml'**: Builds the Docker image and runs the container. It uses environment variables ('CLONE_REPO', 'START_SERVICES') to dynamically control behavior.
- **'provision.sh'**: Executed when the container starts. It clones the repository, builds both applications, and starts them as background services.  
  The final 'tail -f /dev/null' command keeps the container running after initialization.

```yaml
# part1/docker-compose.yml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
      - "59001:59001"
    volumes:
      - ./data:/app/data
      - ./provision.sh:/app/provision.sh
    environment:
      - CLONE_REPO=true
      - START_SERVICES=true
      - REPO_URL=https://...
    command: /bin/bash /app/provision.sh
'''

---

### Ensuring H2 Database Retains Data Between Restarts

To ensure data persistence, a Docker volume is used:

- In 'docker-compose.yml', the line 'volumes: - ./data:/app/data' maps a local directory to the container’s '/app/data'.
- The 'provision.sh' script dynamically creates an 'application.properties' file for Spring Boot, configuring H2 to store its database file in this persistent location ('spring.datasource.url=jdbc:h2:file:/app/data/jpadb').  
  This ensures that even if the container is destroyed and recreated, the database data remains intact.

---

### Part 2: App and Database in a Multi-Container Environment

The goal of Part 2 was to separate the Spring application and the H2 database into two distinct containers.

#### Dockerfile for the H2 Database ('part2/DockerfileDB')

This Dockerfile creates a lightweight container for running H2 in server mode.

- **H2 Installation:** Downloads a specific H2 JAR for consistent versioning.
- **Server Mode:** The  instruction starts the H2 database server, configured to accept remote TCP connections ('-tcp -tcpAllowOthers') on port 9092 .

```Dockerfile
# part2/DockerfileDB
FROM eclipse-temurin:17-jdk-focal
RUN apt-get update && apt-get install -y wget netcat-openbsd

ENV H2_DIR=/opt/h2
ENV H2_DATA_DIR=/opt/h2-data

RUN mkdir -p $H2_DIR && mkdir -p $H2_DATA_DIR
RUN wget https://repo1.maven.org/maven2/com/h2database/h2/2.1.214/h2-2.1.214.jar -O $H2_DIR/h2.jar

WORKDIR $H2_DATA_DIR
EXPOSE 9092

CMD ["java", "-cp", "/opt/h2/h2.jar", "org.h2.tools.Server",
     "-tcp", "-tcpAllowOthers", "-tcpPort", "9092",
     "-baseDir", "/opt/h2-data", "-ifNotExists"]
```

#### Dockerfile for the Spring Application ('part2/DockerfileApp')

- **Stage 1:** Uses 'gradle:8.5.0-jdk17' to install Git, clone the repository, and build the application.
- **Stage 2:** Uses a minimal 'eclipse-temurin:17-jre-focal' base image to run the compiled JAR file, ensuring a smaller and more secure runtime image.

```Dockerfile
# part2/DockerfileApp
# === STAGE 1: Build the application ===
FROM gradle:8.5.0-jdk17 AS builder
RUN apt-get update && apt-get install -y git

WORKDIR /home/gradle/project
ARG REPO_URL
RUN git clone ${REPO_URL} .
WORKDIR /home/gradle/project/CA2-part2/tut-gradle
RUN gradle build -x test

# === STAGE 2: Create the final, lightweight runtime image ===
FROM eclipse-temurin:17-jre-focal
WORKDIR /app
COPY --from=builder /home/gradle/project/CA2-part2/tut-gradle/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

### Orchestration, Networking, and Startup Automation

The 'docker-compose.yml' file defines and connects the two services:

```yaml
# part2/docker-compose.yml
services:
  db:
    build:
      context: .
      dockerfile: DockerfileDB
    container_name: cogsi_h2_database
    volumes:
      - ./db-data:/opt/h2-data
    networks:
      cogsi_network:
        ipv4_address: 172.20.0.10
    healthcheck:
      test: ["CMD-SHELL", "nc -z 172.20.0.10 9092 || exit 1"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s

  app:
    build:
      context: .
      dockerfile: DockerfileApp
      args:
        REPO_URL: https://<TOKEN>@github.com/pedroteixeira80/cogsi2526-1201264-1210901-1210902.git
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:h2:tcp://db:9092/./h2-data/jpadb
      - SPRING_DATASOURCE_USERNAME=sa
      - SPRING_DATASOURCE_PASSWORD=
    networks:
      cogsi_network:
        ipv4_address: 172.20.0.20
    depends_on:
      db:
        condition: service_healthy

networks:
  cogsi_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

#### Notes

- **Configuring H2 in Server Mode:**  
  The 'app' service connects to the database via environment variables.  
  The URL 'jdbc:h2:tcp://db:9092/...' uses the service name 'db', which Docker resolves automatically through its internal network.

- **Automating Startup Order:**  
  The healthcheck ensures the database container is ready before the app starts.  
  Docker Compose waits for the 'db' service to report healthy before launching the 'app'.

- **Security to Restrict Database Access:**  
  The database container does **not** expose port 9092 to the host, only to the internal network, ensuring that only the app container can access it.

- **Resource Management:**  
  No CPU or memory limits were configured. Docker dynamically allocates host resources as needed.
