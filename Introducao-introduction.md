0. Introdução aos conceitos

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

1. Installing Docker on AWS EC2

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

2. Installing Jenkins through Docker

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

Para executar algumas tarefas de IC (tais como testes de aceitação), tive de instalar o Docker no Docker. Assim, entrei no contentor Jenkins, e instalei o Docker dentro do contentor Jenkins, utilizei os comandos seguintes:

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

3. A new Jenkins Item




