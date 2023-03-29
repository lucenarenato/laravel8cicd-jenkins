# Configure Laravel 8 for CI/CD with Jenkins and GitHub

A very basic Laravel 8 application used to be combined with Jenkins and Docker to apply CI/CD in Laravel. 

> Leia um livro chamado “Continuous Delivery with Docker and Jenkins — Second edition” de Rafal LESZKO.
                                    
```sh
❯ docker image build -t laravelcicd .                                           

❯ docker run -d -p 8080:80 --name laravel laravelcicd:latest                                 

❯ docker image build -t jenkins-php .docker 

❯ docker run -d -p 49001:8080 -p 50000:50000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins jenkins-php   

❯ docker exec -it -u root jenkins bash

❯ docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```


[More info here](https://blog.renatolucena.net/post/laravel-ci-cd-jenkins-github)
