# Configurar Laravel 8 para CI/CD com Jenkins e GitHub

## 0. Introdução aos conceitos

Docker

O Docker é uma ferramenta projetada para facilitar a criação, implantação e execução de aplicativos usando contêineres.

Jenkins

Jenkins é uma ferramenta de automação de código aberto escrita em Java com plug-ins criados para fins de Integração Contínua.

GitHub

O GitHub é uma plataforma de hospedagem de código para controle de versão e colaboração.

Continuous integration (CI) and continuous delivery (CD)

A integração contínua (CI) e a entrega contínua (CD) incorporam uma cultura, um conjunto de princípios operacionais e uma coleção de práticas que permitem que as equipes de desenvolvimento de aplicativos forneçam alterações de código com mais frequência e confiabilidade. A implementação também é conhecida como pipeline de CI/CD.

Laravel

O Laravel é um framework web PHP gratuito e de código aberto, criado por Taylor Otwell e destinado ao desenvolvimento de aplicações web seguindo o padrão de arquitetura model-view-controller (MVC) e baseado no Symfony.

Amazon Elastic Compute Cloud (Amazon EC2)

Amazon Elastic Compute Cloud (Amazon EC2) é um serviço web que fornece capacidade de computação segura e escalável na nuvem.

Amazon Relational Database Service (Amazon RDS) 

O Amazon Relational Database Service (Amazon RDS) facilita a configuração, operação e dimensionamento de um banco de dados relacional na nuvem.

Unit testing

O teste de unidade é um método de teste de software pelo qual unidades individuais de código-fonte — conjuntos de um ou mais módulos de programa de computador.

Code coverage

A cobertura de código é uma métrica que pode ajudar você a entender quanto de sua fonte é testada.

Static code analysis 

A análise de código estático é um método de depuração que examina o código-fonte antes que um programa seja executado.

Acceptance Testing

O Teste de Aceitação serve para avaliar se o Produto está funcionando para o usuário, corretamente para o uso.

## 1. Installing Docker on AWS EC2

Utilizo o Amazon AWS EC2 para criar máquinas virtuais / instâncias EC2 (VMs). Criei uma instância EC2 muito simples com as características seguintes:

- Image: Amazon Linux AMI 2018
- Instance: t2.micro
- Security groups: I opened HTTP, HTTPS ports. And I opened TCP 50000, and TCP 49001.

Eu me conectei à minha instância do EC2 por meio do SSH e, em seguida, instalei o Docker na minha instância do EC2:

```sh
sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo usermod -a -G docker ec2-user
```

## 2. Installing Jenkins through Docker

> Então, tive que criar um container para fazer o deploy do Jenkins. Na primeira vez, criei um contêiner simples com o comando docker run -d -p 49001:8080 -p 50000:50000 -v $HOME/jenkins_home:/var/jenkins_home -- name jenkins jenkins/jenkins (NÃO use este comando, este é apenas um exemplo). E comecei a usar Jenkins.

> O livro mencionou que você precisa ter um mestre Jenkins e alguns escravos Jenkins. Para simplificar, decidi ter apenas um container Jenkins, rodando como master e também como slave (rodando jobs). Para fazer isso, tive que criar um Dockerfile com Jenkins, mas também com PHP 7.4 (para rodar Laravel), Composer e outros pacotes. O Dockerfile final é apresentado a seguir:

```sh
FROM jenkins/jenkins:2.257-centos7
USER root
RUN yum install epel-release -y
RUN rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
RUN yum --enablerepo=remi-php74 install php php-mbstring php-xml php-pdo php-pdo_mysql php-xdebug -y
RUN yum update -y 
RUN cd /tmp
RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer
```

Em seguida, criei uma nova imagem do Docker (chamei-a de jenkins-php) e executei um contêiner docker:

```sh
docker image build -t jenkins-php .
docker run -d -p 49001:8080 -p 50000:50000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins jenkins-php
```

Para executar algumas tarefas de IC (tais como testes de aceitação), tive de instalar o Docker no Docker. Assim, entrei no container Jenkins, e instalei o Docker dentro do container Jenkins, utilizei os comandos seguintes:

```sh
docker exec -it -u root jenkins bash
yum install -y yum-utils
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
docker --version
docker run hello-world
chmod 666 /var/run/docker.sock
exit
```

Então eu usei a próxima linha, para pegar a senha inicial do Jenkins (para configurar o Jenkins no navegador)

```sh
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Em seguida, abri minha instância do AWS EC2 no navegador (Acessando a porta 49001). Algo assim: http://EC2DNS.amazonaws.com:49001/

Instalei os plugins recomendados e instalei outro plugin chamado “GitHub integration”

## 3. A new Jenkins Item

Antes de criar um novo artigo Jenkins, desenhei uma aplicação Laravel 8 muito básica, que pode ser encontrada aqui: https://github.com/lucenarenato/laravel8cicd-jenkins

<p align="center"><img src="public/img/1.webp" width="600"></p>

Então, criei um novo item, chamei de “laravel” e selecionei a opção “Pipeline”.

Selecionei o “Pipeline script from SCM”, escolho “Git” e coloquei o endereço do meu projeto GitHub https://github.com/lucenarenato/laravel8cicd-jenkins.git — você pode usar esse endereço se quiser, ou pode usar o Fork meu projeto GitHub.

Esse projeto já contém um arquivo Jenkins com todos os estágios do pipeline. Nas próximas seções, explicarei a maioria desses estágios. Incluindo o que fiz para: (i) executar testes de unidade, (ii) executar teste de cobertura de código, (iii) executar testes de código estático e (iv) executar testes de aceitação. A próxima figura mostra o que eu implementei.

Pipeline Laravel

<p align="center"><img src="public/img/2.webp" width="600"></p>

## 4. Executando testes de unidade

Para executar testes de unidade/recurso automaticamente, crio um teste simples em meu projeto Laravel. Esse teste pode ser encontrado aqui: /test/Feature/RouteTest.php

```php

<?php
namespace Testes\Recurso;
use Illuminate\Foundation\Testing\RefreshDatabase; 
use Illuminate\Fundação\Teste\ComFaker; 
use Testes\TestCase;
class RouteTest extends TestCase 
{ 
    public function testHome() 
    { 
        $this->get('/') 
            ->assertSee('Home'); 
    }
public function testLogin() 
    { 
        $this->get('/login') 
            ->assertSee('Remember Me'); 
    } 
}
```

Esse teste verifica se a rota “/” exibe um texto “Home”. E a rota “/login” exibe um texto “Lembre-se de mim”. Estes são testes muito simples; aqui você pode aplicar alguns testes de classe, testes de modelo, controladores, etc, e assim por diante.

Então desenvolvi uma versão inicial do arquivo Jenkins para executar os testes de unidade:

```json
pipeline {
 agent any
 stages {
        stage("Build") {
            steps {
                sh 'php --version'
                sh 'composer install'
                sh 'composer --version'
                sh 'cp .env.example .env'
                sh 'php artisan key:generate'
            }
        }
        stage("Unit test") {
            steps {
                sh 'php artisan test'
            }
        }
  }
}
```

O código anterior compila o projeto e executa o comando “PHP craftstest” que executa os testes unitários (esse comando executa PHPUnit /um framework PHP para testes unitários / que já está incluído nos projetos Laravel). Mais informações aqui: https://phpunit.de/

Eu construo o item Laravel (do Jenkins), e ele me mostra uma mensagem dizendo que passou nas etapas “Declarative: Checkout SCM, Build, Unit test”. Jó marcou tudo em verde.

## 5. Versão aprimorada com credenciais do Jenkins

O pipeline anterior não alterou as variáveis ​​.env (nome de usuário do banco de dados, senha, host). Então, usei as credenciais do Jenkins (do painel de administração do Jenkins) e criei quatro credenciais. Mais informações aqui: https://www.jenkins.io/doc/book/using/using-credentials/

Aviso: para simplificar, implementei o banco de dados do projeto no Amazon RDS. Mais informações aqui: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateDBInstance.html

O pipeline de atualização do Jenkins é apresentado:

```json

pipeline {
    agent any
    stages {
        stage("Build") {
            environment {
                DB_HOST = credentials("laravel-host")
                DB_DATABASE = credentials("laravel-database")
                DB_USERNAME = credentials("laravel-user")
                DB_PASSWORD = credentials("laravel-password")
            }
            steps {
                sh 'php --version'
                sh 'composer install'
                sh 'composer --version'
                sh 'cp .env.example .env'
                sh 'echo DB_HOST=${DB_HOST} >> .env'
                sh 'echo DB_USERNAME=${DB_USERNAME} >> .env'
                sh 'echo DB_DATABASE=${DB_DATABASE} >> .env'
                sh 'echo DB_PASSWORD=${DB_PASSWORD} >> .env'
                sh 'php artisan key:generate'
                sh 'cp .env .env.testing'
                sh 'php artisan migrate'
            }
        }
        stage("Unit test") {
            steps {
                sh 'php artisan test'
            }
        }
    }
```

## 6. Executando o teste de cobertura de código

Executar o teste de cobertura de código foi muito simples, PHPUnit já suporta teste de cobertura de código. Atualizei o pipeline do Jenkins e incluí uma nova etapa:

```json
stage("Code coverage") {
            steps {
                sh "vendor/bin/phpunit --coverage-html 'reports/coverage'"
            }
        }
```

Eu construo o item Laravel (do Jenkins), e ele me mostra uma mensagem dizendo que passou nas etapas “Declarative: Checkout SCM, Build, Unit test, Code coverage”. Jó marcou tudo em verde.

O relatório de cobertura de código foi encontrado em uma rota como esta: http://EC2DNS.compute-1.amazonaws.com:49001/job/Laravel/21/execution/node/3/ws/reports/coverage/dashboard.html ( você pode navegar a partir do espaço de trabalho do seu trabalho). Esse relatório mostrou algumas aulas sem testes (0%) e outras com (80%, 50% cobertos). Não implementei um mínimo de restrição de cobertura de código, mas é uma boa ideia definir um mínimo.

## 7. Análise de código estático

Para executar a análise de código estático, instalei dois pacotes no meu projeto Laravel:

LaraStan https://github.com/nunomaduro/larastan
PHP CodeSniffer https://github.com/squizlabs/PHP_CodeSniffer
Para o LaraStan, criei um arquivo phpstan.neon na pasta raiz do projeto com o próximo conteúdo

```json
includes:
 — ./vendor/nunomaduro/larastan/extension.neon
parameters:
paths:
 — app/Http/Controllers/
level: 6
checkMissingIterableValueType: false
```

Basicamente, ele verifica todos os arquivos do meu controlador com algumas regras predefinidas. Mais informações aqui: https://github.com/nunomaduro/larastan — as regras incluídas para verificar comentários válidos antes de cada método, alguns espaços entre colchetes e muito mais.

Também instalei o PHP CodeSniffer e modifiquei o arquivo phpcs.xml na pasta raiz do projeto com o seguinte conteúdo:

```xml
<?xml version=”1.0"?>
<ruleset name=”PSR2"> 
<description>The PSR2 coding standard.</description> 
<rule ref=”PSR2"/> 
<file>app/Http/Controllers/</file> 
<exclude-pattern>vendor</exclude-pattern> 
<exclude-pattern>resources</exclude-pattern> 
<exclude-pattern>database/</exclude-pattern> 
<exclude-pattern>storage/</exclude-pattern> 
<exclude-pattern>node_modules/</exclude-pattern>
</ruleset>
```

Eu só queria verificar os controladores (mas obviamente você pode aplicá-lo a modelos e muitos outros arquivos). Ele usa um padrão de codificação PSR2 que verifica camelCase nos nomes dos métodos, alguns espaços e muito mais. Mais informações aqui: https://www.php-fig.org/psr/psr-2/

Por fim, atualizei o pipeline do Jenkins e incluí duas novas etapas:

```json
stage("Static code analysis larastan") {
            steps {
                sh "vendor/bin/phpstan analyse --memory-limit=2G"
            }
        }
        stage("Static code analysis phpcs") {
            steps {
                sh "vendor/bin/phpcs"
            }
        }
```

Eu construo o item Laravel (do Jenkins), e ele me mostra uma mensagem dizendo que passou nas etapas “Declarativa: Checkout SCM, Build, Teste de unidade, Cobertura de código, Análise de código estático larastan, Análise de código estático phpcs”. Jó marcou tudo em verde.

8. Docker dentro do Docker (executando um container do projeto Laravel)

Docker in Docker foi desafiador, enfrentei muitos problemas, mas no final consegui. Após os testes de código estático, tive que implementar os “testes de aceitação”. O teste de aceitação requer uma versão de teste do aplicativo em execução. E então, você executa alguns testes nessa versão em execução.

Isso significa que você precisa executar alguns comandos, construir uma nova imagem docker (dentro do contêiner Jenkins) com base no código atual do GitHub, executar um novo contêiner e testar esse contêiner.

Criei uma conta Docker Hub, para colocar essas imagens online. E evite criar um registro do Docker. Na parte 2 desta história, já expliquei como instalar o Docker no Docker. Então, a próxima parte foi adicionar dois novos estágios ao pipeline do Jenkins:

```json
stage("Docker build") {
            steps {
                sh "docker rmi cpdrenato/laravel8cicd-jenkins"
                sh "docker build -t cpdrenato/laravel8cicd-jenkinspart2 --no-cache ."
            }
        }
        stage("Docker push") {
            environment {
                DOCKER_USERNAME = credentials("docker-user")
                DOCKER_PASSWORD = credentials("docker-password")
            }
            steps {
                sh "docker login --username ${DOCKER_USERNAME} --password ${DOCKER_PASSWORD}"
                sh "docker push cpdrenato/laravel8cicd-jenkinspart2"
            }
        }
```        

Esses estágios constroem uma nova imagem com base no código atual do GitHub e se conectam ao Docker Hub (através do uso de duas credenciais do Jenkins que registrei) e, em seguida, enviam a nova imagem para minha conta do Docker Hub. Você pode encontrar essa imagem aqui: https://hub.docker.com/r/cpdrenato/laravel8cicd-jenkins

Ter essa imagem online me permite criar novos containers facilmente, e aplicar os testes de aceitação.

9. Testes de aceitação

Eu corro dois tipos de testes de aceitação. Com comando CURL e com pacote de co-decepção.

O comando CURL foi uma primeira tentativa para verificar se eu era capaz de executar um novo contêiner e aplicar alguns testes a esse contêiner em execução.

Criei uma funcionalidade bem simples no laravel para somar dois números, veja o próximo código (que foi adicionado no /app/Http/Controllers/HomeController.php):

```php
    /**
     * Return sumatory
     *
     * @return int
     */
    public function sum(int $num1, int $num2)
    {
        return $num1+$num2;
    }
```

Se você abrir o navegador e acessar /public/sum/4/2, ele exibirá 6.

Em seguida, crio um arquivo accept_test.sh para usar o comando CURL e testar a funcionalidade anterior. Esse arquivo obtém o endereço IP do contêiner em execução e invoca o /public/sum/4/2. Em seguida, compara o resultado com o número “6”. Se ambos forem iguais, o teste passou, caso contrário, apresenta um erro.

```bash
#!/bin/bash
res=$(curl -s -w “%{http_code}” $(docker inspect -f ‘{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}’ laravel8cd)/public/sum/4/2)
body=${res::-3}
if [ $body != “6” ]; then
 echo “Error”
fi
```

Atualizei o pipeline Jenkins com os próximos dois estágios:

```json
        stage("Deploy to staging") {
            steps {
                sh "docker run -d --rm -p 80:80 --name laravel8cicd-jenkins cpdrenato/laravel8cicd-jenkins"
            }
        }
        stage("Acceptance test curl") {
            steps {
                sleep 20
                sh "chmod +x acceptance_test.sh && ./acceptance_test.sh"
            }
            post {
                always {
                    sh "docker stop laravel8cicd-jenkins"
                }
            }
        }
```

Ele executa um contêiner com base na imagem do hub do docker de atualização e, em seguida, executa o arquivo accept_test.sh (mas antes, ele espera 20 segundos para permitir que o docker crie o contêiner com sucesso). No final da etapa e sempre, pára o container.

Por fim, o segundo teste de aceitação foi com o uso do pacote PHP denominado PHP codeception.

Eu instalei PHP codeception https://codeception.com/for/laravel — e criei um teste de aceitação muito simples (que pode ser encontrado em /tests/acceptance/RegisterCest.php):

```php
<?php
class RegisterCest
{
    public function tryToTest(AcceptanceTester $I)
    {
        $I->amOnPage('/');
        $I->click('Register');
        $I->see('Confirm Password');
    }
}
```

Este teste de aceitação verifica se a rota “/” contém um link chamado Register e clica nesse link; portanto, verifica se a nova página carregada contém um texto chamado “Confirmar senha”.

Atualizei o pipeline do Jenkins com o estágio final:

```json
       stage("Acceptance test codeception") {
            steps {
                sh "vendor/bin/codecept run"
            }
            post {
                always {
                    sh "docker stop laravel8cd"
                }
            }
        }
```

Essa etapa executa os testes de codeceptions e, ao final, para o contêiner.

Eu construo o item Laravel (do Jenkins), e ele me mostra uma mensagem dizendo que passou no “Declarativo: Checkout SCM, Build, Teste de unidade, Cobertura de código, Análise de código estático larastan, Análise de código estático phpcs, Docker build, Docker push , Implantar para encenação, Curva de teste de aceitação, Co-engano de teste de aceitação”. Jó marcou tudo em verde.
“Declarative: Checkout SCM, Build, Unit test, Code coverage, Static code analysis larastan, Static code analysis phpcs, Docker build, Docker push, Deploy to staging, Acceptance test curl, Acceptance test co-deception”

O resultado final é apresentado na Figura 2.

Resumo

Esta história explica como aplicar alguns princípios de CI/CD em projetos Laravel (com o uso de Jenkins e Docker). O exemplo foi muito básico para introduzir alguns conceitos principais. Mas você deve considerar que existem muitos riscos com o exemplo apresentado. Os escravos Jenkins devem ser separados. Nem sempre é possível ter o banco de dados no Amazon RDS, alguns autores não recomendam o uso do Docker no Docker e assim por diante. No entanto, para desenvolvedores iniciantes de Laravel CI/CD (como eu), acho que esta é uma boa introdução.

# Continuação ...

Vou apresentar como finalizar o processo de CI/CD (com base em alguns elementos que tirei dos capítulos 6 a 8 do livro Continuous Delivery). Neste caso, faremos algumas modificações no código apresentado na parte 1, e iremos desenhar e implementar um pipeline completo, que nos permitirá lançar as novas versões do projeto Laravel em ambiente de production. O processo completo pode ser encontrado na Figura 1.

<p align="center"><img src="public/img/3.webp" width="600"></p>

## 0. Introdução de novos conceitos

Ansible é a maneira mais simples de automatizar aplicativos e infraestrutura de TI. Implantação de aplicativos + gerenciamento de configuração + entrega contínua.

Os testes de fumaça são um subconjunto de casos de teste que cobrem a funcionalidade mais importante de um componente ou sistema, usados ​​para auxiliar na avaliação de se as principais funções do software parecem funcionar corretamente. Ele determina se a compilação implantada é estável ou não.

Criando instâncias necessárias do AWS EC2
Conforme visto na Figura 1, precisaremos de 3 instâncias diferentes.

VM-Jenkins: será a instância principal, aqui vamos instalar o Docker, e vamos fazer o deploy de um container Jenkins que internamente vai incluir: Docker (Docker dentro do Docker), PHP e Ansible.

Eu criei essa instância com base nas seguintes características:

- Imagem: Amazon Linux AMI 2018
- Instância: t2.micro
- Grupos de segurança: abri portas HTTP, HTTPS. E abri o TCP 50000 e o TCP 49001.


VM-Staging: representará o ambiente de staging. Vamos usá-lo para implantar as novas versões do projeto e aplicar alguns testes de aceitação. Para esta instância, adicionei uma nova tag (através do painel AWS EC2) com o próximo conteúdo (chave: “type”, valor: “staging”).

VM-Production: representará o ambiente de production. Vamos usá-lo para implantar as novas versões do projeto e aplicar alguns testes de fumaça. Para esta instância, adicionei uma nova tag (através do painel AWS EC2) com o próximo conteúdo (chave: “tipo”, valor: “production”).

Criei as duas instâncias anteriores com base nas seguintes características:

- Imagem: Amazon Linux 2 AMI
- Instância: t2.micro
- Grupos de segurança: abri portas HTTP, HTTPS.


<p align="center"><img src="public/img/4.webp" width="600"></p>

## 1. Configurando a instância do Jenkins
Eu instalei o Docker nessa instância:

```sh
sudo yum update -y 
sudo yum install -y docker 
sudo service docker start 
sudo usermod -a -G docker ec2-user
```

## Criei o mesmo Dockerfile 

```sh
FROM jenkins/jenkins:2.257-centos7
USER root
RUN yum install epel-release -y
RUN rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
RUN yum — enablerepo=remi-php74 install php php-mbstring php-xml php-pdo php-pdo_mysql php-xdebug -y
RUN yum update -y 
RUN cd /tmp
RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer
```

Em seguida, criei uma imagem do docker (chamei-a de jenkins-php) e executei um contêiner docker:

```sh
docker image build -t jenkins-php .docker run -d -p 49001:8080 -p 50000:50000 \
 -v /var/run/docker.sock:/var/run/docker.sock \
 — name jenkins jenkins-php
```

A diferença com o container Jenkins (relativamente à parte 1 da história), foi que, neste caso, terei de instalar Docker dentro do Docker, mas também Ansible. Ansible será utilizado para ligar e executar alguns comandos dentro das instâncias de encenação e production. Para isso, executei os comandos seguintes:

```sh
docker exec -it -u root jenkins bash
yum install -y yum-utils
yum-config-manager \
 — add-repo \
 https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
docker — version
chmod 666 /var/run/docker.sock
yum install ansible -y
ansible --version
yum install -y nano
```

Instalei o Docker, mas também instalei o Ansible. Apliquei a próxima configuração no Ansible.

```sh
cd /etc/ansible/
nano ansible.cfg
```

Eu incluí as próximas 3 linhas após a linha [defaults]

```sh
inventory = ./ansible_plugins
enable_plugins = aws_ec2
host_key_checking = False
```

Depois, crio uma nova pasta e crio um ficheiro yml que representa o inventário dinâmico possível (o código seguinte permite-me recolher dinamicamente a minha informação de instâncias EC2). Mais informações aqui: https://aws.amazon.com/es/blogs/apn/getting-started-with-ansible-and-dynamic-amazon-ec2-inventory-management/

```sh
mkdir ansible_plugins
cd ansible_plugins
nano production_aws_ec2.yml
```

Coloquei o próximo conteúdo no arquivo anterior (troquei o “aws_access_key” e “aws_secret_key” para minhas respectivas chaves):

```
# aws ec2 ansible dynamic inventory plugin
plugin: aws_ec2
#set aws_access_key and secret_key.
aws_access_key: AWS_ACCESS_KEY
aws_secret_key: AWS_SECRET_KEY
# set the regions. 
regions: 
  - us-east-1
# - us-east-2
# set strict to False    
# if True this will make invalid entries 
# a fatal error
strict: False
keyed_groups:
  #  each aws ec2 instance has it own instance tags. create  
  #  a tag variable from those tags for ansible to use. 
  #  if the ec2 tag Name had the value cygnusx1 the tag 
  #  variable would be: 
  #  tag_Name_cygnusx1
  #  if a tag existed for an aws instance as  
  #  Applications with the value of Oracle the  
  #  variable would be:
  #  tag_Applications_Oracle
  - key: tags
    prefix: tag
  #
  # the following keyed groups are from the aws url:
  # https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html#options   
  # below are some of the variable that can be used.  
  # an example for instance_type: 
  # aws_instance_type_t2_micro
  - key: architecture
    prefix: arch
  - key: tags.Applications
    separator: ''
  - key: instance_type
    prefix: aws_instance_type
  - key: placement.region
    prefix: aws_region
  - key: image_id
    prefix: aws_image
  - key: hypervisor
    prefix: aws_hypervisor
  - key: 'security_groups|json_query("[].group_id")'
    prefix: 'security_groups'
hostnames:
# a list in order of precedence for hostname variables.
# 
  - ip-address
  - dns-name
  - tag:Name
  - private-ip-address
compose:
# use if you need to connect via the ec2
# private ip address. 
#
# this is needed for example in a 
# corporate / company environment where ec2 
# instances don't use a public ip address
#
#  ansible_host: private_ip_address
```

Também instalei duas bibliotecas requeridas pelo Ansible, e crio um novo ficheiro para guardar a minha chave pem. Este ficheiro será utilizado mais tarde na conduta Jenkins (para nos permitir ligar a instância Jenkins com as instâncias de encenação e production).

```sh
yum -y install python-pip
pip install boto3
cd /etc/ansible
mkdir pem
nano pem/key.pem
```

Dentro do arquivo anterior, coloquei meu conteúdo key.pem (que normalmente uso para conectar minhas instâncias via SSH). Algo assim:

```pem
-----BEGIN RSA PRIVATE KEY-----
MIIE…
-----END RSA PRIVATE KEY-----
```

## 2. Criando os playbooks do Ansible

Trabalhar com os ambientes de preparação e production requer comunicação entre o contêiner Jenkins e os ambientes anteriores. Por exemplo: (i) implantar a nova versão no ambiente de preparação (para aplicar os testes de aceitação), ou (ii) implantar o produto lançado no ambiente de production.

Para conseguir realizar os processos anteriores criei 4 playbooks Ansible.

playbook-staging-run.yml

Este playbook permite instalar o software necessário dentro do ambiente de staging e, ao final, executar uma nova versão do projeto laravel (executar um novo contêiner).

```sh
cd /etc/ansible/
mkdir playbook
nano /etc/ansible/playbook/playbook-staging-run.yml
```

Coloco o próximo conteúdo (verifique se corre um novo container cpdrenato/laravel8cicd-jenkins no porto 80 - apenas sobre a instância com tipo de etiqueta: "encenação"):


```
- hosts: tag_type_staging
  become: yes
  remote_user: ec2-user
  name: Upgrade all packages
  tasks: 
    - name: Update all
      yum: name=* state=latest
    - name: install docker
      yum: name=docker state=latest
    - name: Start docker service
      systemd:
        state: started
        name: docker
    - name: Add admin user
      user:
        name: admin
        groups: docker
        append: yes
    - name: install python-pip
      yum: name=python-pip state=present
    - name: install docker-py
      pip: name=docker-py
    - name: Run laravel container
      docker_container: 
        name: laravel8cicd-jenkins 
        image: cpdrenato/laravel8cicd-jenkins 
        state: started 
        exposed_ports: "80"
        published_ports: "80:80"
```

playbook-staging-acceptance.yml

> Este manual permite executar o teste de aceitação sobre o recipiente em funcionamento no ambiente de encenação.

`nano /etc/ansible/playbook/playbook-staging-acceptance.yml`

Eu coloco o próximo conteúdo (verifique se ele executa o teste de aceitação de codeception s— somente sobre a instância com tipo de tag: “staging”):

```
- hosts: tag_type_staging
  become: yes
  remote_user: ec2-user
  tasks: 
    - name: Run acceptance tests
      command: docker exec -it laravel8cicd-jenkins bash -c 'vendor/bin/codecept run';
      register: atesting
    - debug: msg="{{ atesting.stdout }}"
```

playbook-staging-stop.yml

Este playbook permite parar o contêiner e remover a imagem do docker.
`nano /etc/ansible/playbook/playbook-staging-stop.yml`

## Eu coloquei o seguinte conteúdo:

```
- hosts: tag_type_staging
  become: yes
  remote_user: ec2-user
  tasks: 
    - name: Delete laravel containers
      docker_container:
        name: laravel8cicd-jenkins
        force_kill: true
        keep_volumes: false
        state: absent
     - name: Remove image
      docker_image:
        state: absent
        name: cpdrenato/laravel8cicd-jenkins
```

playbook-production-run.yml

Este manual permite instalar o software necessário dentro do ambiente de production, e no final, executa uma nova versão do projecto laravel (corre um novo container).


`nano /etc/ansible/playbook/playbook-production-run.yml`

Eu coloquei o próximo conteúdo (verifique se ele executa um novo contêiner cpdrenato/laravel8cicd-jenkins na porta 80 — apenas sobre a instância com o tipo de tag: “production”):

```
- hosts: tag_type_production
  become: yes
  remote_user: ec2-user
  name: Upgrade all packages
  tasks: 
    - name: Update all
      yum: name=* state=latest
    - name: install docker
      yum: name=docker state=latest
    - name: Start docker service
      systemd:
        state: started
        name: docker
    - name: Add admin user
      user:
        name: admin
        groups: docker
        append: yes
    - name: install python-pip
      yum: name=python-pip state=present
    - name: install docker-py
      pip: name=docker-py
    - name: Delete laravel containers
      docker_container:
        name: laravel8cicd-jenkins
        force_kill: true
        keep_volumes: false
        state: absent
    - name: Remove image
      docker_image:
        state: absent
        name: cpdrenato/laravel8cicd-jenkins
    - name: Run laravel container
      docker_container: 
        name: laravel8cicd-jenkins 
        image: cpdrenato/laravel8cicd-jenkins 
        state: started 
        exposed_ports: "80"
        published_ports: "80:80"
```

playbook-production-acceptance.yml

Este livro de jogo permite executar os testes de fumo sobre o recipiente em funcionamento no ambiente de production.

`nano /etc/ansible/playbook/playbook-production-acceptance.yml`

Eu coloco o próximo conteúdo (verifique se ele executa o teste de aceitação de codeception s — somente sobre a instância com tipo de tag: “production”):


```
- hosts: tag_type_production
  become: yes
  remote_user: ec2-user
  tasks: 
    - name: Run smoke tests
      command: docker exec -it laravel8cicd-jenkins bash -c 'vendor/bin/codecept run';
      register: stesting
    - debug: msg="{{ stesting.stdout }}"
```

Neste caso, os testes de aceitação e os testes de fumaça são os mesmos. No entanto, comumente, os testes de fumaça são um subconjunto dos testes de aceitação.

## 3. Canalização completa

Desenvolvi um novo projeto GitHub sobre essa história. Esse projeto GitHub pode ser encontrado aqui: https://github.com/cpdrenato/laravel8cicd-jenkins

A última parte foi atualizar o arquivo jenkins, para ficar consistente com os novos ambientes e as novas tarefas. O arquivo jenkins completo é apresentado a seguir:

```json
pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID = credentials("aws-access-key-id")
        AWS_SECRET_ACCESS_KEY = credentials("aws-secret-access-key")
        AWS_SESSION_TOKEN = credentials("aws-session-token")
    }
    stages {
        stage("Build") {
            environment {
                DB_HOST = credentials("laravel-host")
                DB_DATABASE = credentials("laravel-database")
                DB_USERNAME = credentials("laravel-user")
                DB_PASSWORD = credentials("laravel-password")
            }
            steps {
                sh 'php --version'
                sh 'composer install'
                sh 'composer --version'
                sh 'cp .env.example .env'
                sh 'echo DB_HOST=${DB_HOST} >> .env'
                sh 'echo DB_USERNAME=${DB_USERNAME} >> .env'
                sh 'echo DB_DATABASE=${DB_DATABASE} >> .env'
                sh 'echo DB_PASSWORD=${DB_PASSWORD} >> .env'
                sh 'php artisan key:generate'
                sh 'cp .env .env.testing'
                sh 'php artisan migrate'
            }
        }
        stage("Unit test") {
            steps {
                sh 'php artisan test'
            }
        }
        stage("Code coverage") {
            steps {
                sh "vendor/bin/phpunit --coverage-html 'reports/coverage'"
            }
        }
        stage("Static code analysis larastan") {
            steps {
                sh "vendor/bin/phpstan analyse --memory-limit=2G"
            }
        }
        stage("Static code analysis phpcs") {
            steps {
                sh "vendor/bin/phpcs"
            }
        }
        stage("Docker build") {
            steps {
                sh "docker rmi cpdrenato/laravel8cicd-jenkins"
                sh "docker build -t cpdrenato/laravel8cicd-jenkins --no-cache ."
            }
        }
        stage("Docker push") {
            environment {
                DOCKER_USERNAME = credentials("docker-user")
                DOCKER_PASSWORD = credentials("docker-password")
            }
            steps {
                sh "docker login --username ${DOCKER_USERNAME} --password ${DOCKER_PASSWORD}"
                sh "docker push cpdrenato/laravel8cicd-jenkins"
            }
        }
        stage("Deploy to staging") {
            steps {
                sh "export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}"
                sh "export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}"
                sh "export AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN}"
                sh "ssh-agent sh -c 'ssh-add /etc/ansible/pem/key.pem && ansible-playbook /etc/ansible/playbook/playbook-staging-run.yml'"
            }
        }
        stage("Acceptance test curl") {
            steps {
                sleep 20
                sh "chmod +x acceptance_test.sh && ./acceptance_test.sh"
            }
        }
        stage("Acceptance test codeception") {
            steps {
                sh "export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}"
                sh "export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}"
                sh "export AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN}"
                sh "ssh-agent sh -c 'ssh-add /etc/ansible/pem/key.pem && ansible-playbook /etc/ansible/playbook/playbook-staging-acceptance.yml'"
            }
            post {
                always {
                    sh "export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}"
                    sh "export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}"
                    sh "export AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN}"
                    sh "ssh-agent sh -c 'ssh-add /etc/ansible/pem/key.pem && ansible-playbook /etc/ansible/playbook/playbook-staging-stop.yml'"
                }
            }
        }
        stage("Release") {
            steps {
                sh "export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}"
                sh "export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}"
                sh "export AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN}"
                sh "ssh-agent sh -c 'ssh-add /etc/ansible/pem/key.pem && ansible-playbook /etc/ansible/playbook/playbook-production-run.yml'"
            }
        }
        stage("Smoke test") {
            steps {
                sleep 20
                sh "export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}"
                sh "export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}"
                sh "export AWS_SESSION_TOKEN=${AWS_SESSION_TOKEN}"
                sh "ssh-agent sh -c 'ssh-add /etc/ansible/pem/key.pem && ansible-playbook /etc/ansible/playbook/playbook-production-acceptance.yml'"
            }
        }
    }
}
```

Existem algumas diferenças entre este pipeline e o pipeline apresentado na parte 1.

- As credenciais da AWS foram adicionadas (para se conectar com as outras instâncias).
- “Deploying to staging”: agora não implanta a nova versão do projeto dentro do container jenkins. Agora, ele implanta a nova versão do projeto dentro do ambiente de preparação.
- “Acceptance test curl” e “Acceptance test codeception”: agora eles rodam os testes sobre o container dentro do ambiente de staging.
- “Release”: implanta a nova versão do projeto dentro do ambiente de produção.
- “Smoke test”: executa os testes de fumaça sobre o container dentro do ambiente de produção.

## 4. Item de Jenkins

Eu crio um novo item jenkins (chamado “laravel8cicd-jenkins”), conectei-o ao meu repo: https://github.com/cpdrenato/laravel8cicd-jenkins e construí um novo job. A Figura 3 mostra os resultados.


<p align="center"><img src="public/img/5.webp" width="600"></p>

Depois que o trabalho foi concluído. Verifiquei o DNS da minha instância e vi o projeto carregando corretamente (consulte a Figura 4). Isso significa que o processo completo de CI/CD (apresentado na Figura 1) funcionou conforme o esperado).

<p align="center"><img src="public/img/6.webp" width="600"></p>

Também apliquei uma mudança no projeto laravel. Modifiquei o texto da página principal, de “Home” para “Home — V2”. Em seguida, um novo trabalho foi iniciado automaticamente (graças ao plug-in de integração jenkins GitHub) e 4 minutos depois recarreguei o URL da instância de produção e a próxima versão do projeto foi carregada corretamente (verifique a Figura 5).

<p align="center"><img src="public/img/7.webp" width="600"></p>

## 5. Notas importantes

Lembre-se que este exemplo é muito básico e há coisas importantes a considerar:

- Dimensionamento: você deve aplicar Docker Swarm ou Kubernetes se quiser dimensionar seus aplicativos. Este exemplo mostra apenas 3 VMs sem escalonamento.
- Rollbacks de implantação: deve haver um sistema para reverter para uma versão anterior em caso de falhas.
- Versionamento: você deve incluir um sistema de versionamento para gerenciar suas imagens. Eu não o apliquei neste exemplo básico. Mas você pode encontrar mais informações na internet ou no livro que recomendo.
- Outros testes: você deve incluir outros tipos de testes em seu pipeline. Tais como: testes de segurança, testes de desempenho e muito mais.
- Segurança do hub do Docker: minhas imagens do hub do Docker são públicas e já estão conectadas ao meu RDS (que foi excluído). Portanto, lembre-se de colocá-los em sigilo ou, pelo menos, criar as imagens sem informações confidenciais.
- Pequenas correções: no arquivo “acceptance_test.sh”, você deve substituir o DNS da minha instância. E lembre-se das sugestões que fiz na parte 1 desta história.
Resumo

> Concluímos uma introdução prática ao CI/CD com Jenkins, Docker, Ansible e com base em projetos Laravel 8. Espero que tenham gostado desta história e adquirido algum conhecimento. Se você tiver alguma dúvida ou sugestão, deixe-a na seção de comentários.

## Fonte:
- https://blog.renatolucena.net/post/laravel-ci-cd-jenkins-github
- https://faun.pub/configure-laravel-8-for-ci-cd-with-jenkins-and-github-part-1-58b9be304292
- https://faun.pub/configure-laravel-8-for-ci-cd-with-jenkins-and-github-part-2-80d9c337f0e4
