# Configurando Laravel 6, Nginx e PostgreSQL com Docker

## Repositório de exemplo do [tutórial](https://medium.com/@vhsilva.ap/configurando-laravel-6-nginx-e-postgresql-com-docker-9ad29c53d5)

![Laravel + Docker](https://miro.medium.com/max/800/1*lesSI3pwzOlVxGve32sazw.jpeg)


Esse é um tutorial passo a passo para configurar **Laravel** em container Docker junto com **Postgres** e **Nginx**.

Nesse tutorial você irá:

-   Criar o  [Dockerfile](https://docs.docker.com/engine/reference/builder/)  da sua aplicação  [Laravel](https://laravel.com/).
-   Criar o  [docker-compose.yml](https://docs.docker.com/compose/)  da sua aplicação e demais dependências.

Para começar vamos criar um projeto novo Laravel utilizando o  [**_composer create-project_**](https://getcomposer.org/doc/03-cli.md#create-project).

# Criando projeto Laravel

Abra seu terminal e execute o comando:

  ```sh
  composer create-project —prefer-dist laravel/laravel blog
  ```

Você pode encontrar outras maneiras de fazer a instalação e iniciar o seu projeto na  [documentação](https://laravel.com/docs/5.5/installation).

# Criando o Dockerfile da aplicação

> O Docker pode criar imagens automaticamente, lendo as instruções de um arquivo Docker chamado Dockerfile é um documento de texto que contém todos os comandos que um usuário pode chamar na linha de comando para montar uma imagem.Usando o docker build, os usuários podem criar um build automatizado que executa várias instruções da linha de comando em sucessão. Fonte: [https://docs.docker.com/engine/reference/builder/](http://O%20Docker%20pode%20criar%20imagens%20automaticamente,%20lendo%20as%20instru%C3%A7%C3%B5es%20de%20um%20arquivo%20Docker%20%20chamado%20Dockerfile%20%C3%A9%20um%20documento%20de%20texto%20que%20cont%C3%A9m%20todos%20os%20comandos%20que%20um%20usu%C3%A1rio%20pode%20chamar%20na%20linha%20de%20comando%20para%20montar%20uma%20imagem.%20%20Usando%20o%20docker%20build,%20os%20usu%C3%A1rios%20podem%20criar%20um%20build%20automatizado%20que%20executa%20v%C3%A1rias%20instru%C3%A7%C3%B5es%20da%20linha%20de%20comando%20em%20sucess%C3%A3o.%20Fonte:%20https//docs.docker.com/engine/reference/builder/)

Então vamos criar o  **Dockerfile**  da nossa aplicação, Abra o projeto no editor de sua preferência e  **crie**  um  **arquivo**  com o  **chamado**  [**```Dockerfile```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/blob/master/Dockerfile).

Com o arquivo criado vamos fazer as configurações necessárias:

Primeiramente vamos adicionar a  [imagem do php 7.3](https://hub.docker.com/_/php)  usaremos uma  [imagem alpine](https://alpinelinux.org/)  por ser mais leve:

```Dockerfile
FROM php:7.3.6-fpm-alpine3.9
```
Além do  **php**  iremos necessitar de outros programas então usando o comando  **apk**  das imagens  **alpine**  vamos adicionar outros software como  [**nodejs**](https://nodejs.org/en/)  e outros:
```Dockerfile
RUN apk add —no-cache openssl bash nodejs npm postgresql-dev
```
Para facilitar nossa vida na hora de instalar extensões do php existe o comando  **docker-php-ext-install**  vamos utilizá-lo instalar extensões como  [**bcmath**](https://www.php.net/manual/pt_BR/book.bc.php)  e  [**pdo**](https://www.php.net/manual/pt_BR/book.pdo.php).
```Dockerfile
RUN docker-php-ext-install bcmath pdo pdo_pgsql
```

Agora vamos definir qual será o nosso diretório principal de trabalho no nosso  [container docker](https://www.docker.com/resources/what-container)  iremos usar a pasta  **```/var/www```**

```Dockerfile
WORKDIR /var/www
```

Como vamos utilizar o Nginx como nosso servidor vamos usar um pequeno “**hack”**, vamos remover a pasta  **```html/```**  de dentro de  **```www/```**, e criar um link simbólico da pasta  **```public/```**  do projeto Laravel apontando para  **```html/```**, assim quando o  **Nginx**  tentar acessar  **```html/```**  irá cair em  **```public/```**  do nosso projeto (sacou a ideia?):
```Dockerfile
RUN rm -rf /var/www/html  
RUN ln -s public html
```

Como iremos utilizar o  [**Composer**](https://getcomposer.org/)  vamos fazer sua instalação também:

```Dockerfile
RUN curl -sS [https://getcomposer.org/installer](https://getcomposer.org/installer) | php — — install-dir=/usr/local/bin — filename=composer
```

Vamos copiar nossa aplicação para o nosso container:
```Dockerfile
COPY . /var/www
```
Não esquecer de dar aquele  **chmod 777**  na pasta  **```storage/```**  do Laravel. (eu sei 777 é um abuso e tals… não me julguem estamos falando de um container isolado):
```Dockerfile
RUN chmod -R 777 /var/www/storage
```
Vamos agora deixar exposta a nossa  **porta ``:9000``**  para ser acessada (Você de tá se perguntando por que a porta  **```:9000```**  e não a **:8000**, bem vamos deixar a  **```:8000```**  para mapear para porta **```:80```** do nosso  **Nginx**  para evitar confusão vamos usar aqui a  **```:9000```**):
```Dockerfile
EXPOSE 9000
```
E vamos usar outro  **“hack”**  para que nosso container continue em execução usando o  **```ENTRYPOINT```**, para manter o processo php ativo:
```Dockerfile
ENTRYPOINT [ “php-fpm” ]
```
Ao final seu arquivo deve estar assim:
```Dockerfile
FROM php:7.3.6-fpm-alpine3.9
RUN apk add --no-cache openssl bash nodejs npm postgresql-dev
RUN docker-php-ext-install bcmath pdo pdo_pgsql

WORKDIR /var/www

RUN rm -rf /var/www/html
RUN ln -s public html


RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer


COPY . /var/www

RUN chmod -R 777 /var/www/storage

EXPOSE 9000

ENTRYPOINT [ "php-fpm" ]
```
### Criando do docker-compose.yml

> O Compose (não confundir com Composer do php) é uma ferramenta para definir e executar aplicativos Docker de vários contêineres. Com o Compose, você usa um arquivo YAML para configurar os serviços do seu aplicativo. Em seguida, com um único comando, você cria e inicia todos os serviços da sua configuração. Fonte: [https://docs.docker.com/compose/](https://docs.docker.com/compose/)

Agora na sua aplicação crie um arquivo chamado: [**```./docker-compose.yml```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/blob/master/docker-compose.yml), vamos definir quem irá executar junto de nossa aplicação.

Vamos usar a [**versão 3**](https://docs.docker.com/compose/compose-file/) da notação do arquivo então inicialmente vamos configurar nossa aplicação:
```yml
version: ‘3’  
services:  
  web:  
  restart: always  
  build: .  
  volumes:  
    - ./:/var/www/
```
Não esquecer que nosso arquivo deve manter a indentação correta por ser tratar de um [YAML](https://yaml.org/)).

#### Entendo a nossa anotação:

1.  Logo após indicar a versão da notação do arquivo vamos listar os serviços. E definimos nosso primeiro serviço como **web;**
2.  Definimos que ele sempre irá reiniciar em (restart:always);
3.  Definimos onde iremos encontrar o **```Dockerfile```** com a configurações da aplicação apontando para pasta local em: **```build: .```**
4.  Por fim definimos o [**volume**](https://docs.docker.com/storage/volumes/) de nossa aplicação, isto irá dizer para o docker ficar de olho em nossa pasta local, e ao mesmo tempo “**linkar**” com a pasta **``/var/www/``** do nosso container assim tudo que for modificado em nossa pasta do projeto será atualizado no container e vice-versa.

### Configurando PostgreSQL

No nosso arquivo [**```./docker-compose.yml```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/blob/master/docker-compose.yml) vamos adicionar o segundo serviço que agora será o nosso banco de dados então após as configurações de **web** adicione o do **postgreSQL**:
```yml
  db:  
    image: postgres:12.0-alpine  
    restart: always  
    environment:  
      POSTGRES_PASSWORD: postgres  
      POSTGRES_DB: blog  
    volumes:  
      - “./.docker/dbdata:/var/lib/postgresql/data”
```
#### Entendo a nossa anotação:

1.  Definimos nosso segundo serviço como **db;**
2.  Indicamos que ele deve incluir a [imagem do postgres](https://hub.docker.com/_/postgres) usaremos uma imagem [alpine](https://alpinelinux.org/) por ser mais leve;
3.  Assim com o primeiro definimos que ele sempre irá reiniciar em (**```restart:always```**);
4.  Agora definimos nossas variáveis de ambiente: a primeira: **```POSTGRES_PASSWORD```** de fine a senha do nosso banco como **```postgres```** (sim suuuper difícil…) e segunda **```POSTGRES_DB```** define o nome do nosso banco de dados como **```blog```** , poderíamos definir outras [variáveis para o postgres](https://hub.docker.com/_/postgres) como **```POSTGRES_USER```** para o nome de usuário do banco, como não foi definida no arquivo o nome será **```postgres```** , assim com a porta de que não foi definida no arquivo então será a porta padrão **```:5432```**.
5.  E por fim definimos o [**volume**](https://docs.docker.com/storage/volumes/) do nosso banco, para não perdermos os dados do nosso banco toda vez iniciarmos nosso container, assim crie uma pasta no seu projeto chamada **```.docker/```** e dentro dela a subpasta **```dbdata/.```**

> **IMPORTANTE!!!** não esquecer de adicionar essa pasta no [**.gitignore**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/blob/master/.docker/.gitignore) pra não tá versionando seu bando à toa!

Assim salvamos nessa pasta salvamos dos dados que estão em **```/var/lib/postgresql/```** data do nosso container que é o padrão onde o **PostgreSQL** salva seus dados.

### Configurando Nginx

Para configurar o nosso servidor vamos fazer algumas configurações prévias.:

Primeiramente na pasta [**```.docker/```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/tree/master/.docker) crie uma subpasta chamada [**```nginx/```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/tree/master/.docker/nginx) e dentro da mesma crie um arquivo chamado [**```nginx.conf```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/blob/master/.docker/nginx/nginx.conf) neste arquivo vamos inserir as seguintes configurações:

```nginx
server {
    listen 80;
    root /var/www/public/;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.html index.htm index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass web:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Esse arquivo é semelhante ao [arquivo de exemplo da documentação do Laravel](https://laravel.com/docs/6.x/deployment#nginx) com uma diferença: em **```fastcgi_pass```** estou apontando para o serviço **```web```** na porta **```:9000```**, que foi porta que definimos para ser exposta.

Agora dentro de **```./.docker/nginx/```** crie um arquivo [**```Dockerfile```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/blob/master/.docker/nginx/Dockerfile) (sim iremos criar um **```Dockerfile```** para configurar nosso **Nginx**).

No arquivo **```Dockerfile```** do **Nginx** adicione a [imagem do nginx](https://hub.docker.com/_/nginx) também use [alpine](https://alpinelinux.org/).
```Dockerfile
FROM nginx:1.15.0-alpine
```
Agora execute os comandos para atualizar os dados na sua imagem e instale o **bash** para interagirmos futuramente com o container.
```Dockerfile
RUN apk update && apk add bash
```

Remova o arquivo de configuração padrão do **nginx**.
```Dockerfile
RUN rm /etc/nginx/conf.d/default.conf
```

Copie nosso arquivo de configuração para o container:
```Dockerfile
COPY ./nginx.conf /etc/nginx/conf.d
```
Nosso **Dockerfile** do **nginx** deve estar assim:

```Dockerfile
FROM nginx:1.15.0-alpine

RUN apk update && apk add bash

RUN rm /etc/nginx/conf.d/default.conf

COPY ./nginx.conf /etc/nginx/conf.d
```
Agora vamos voltar ao nosso arquivo **```docker-compose.yml```** e vamos adicionar o serviço do **nginx**:
```yml
  nginx:  
    build: ./.docker/nginx  
    restart: always  
    ports:  
      - “8000:80”  
    volumes:  
      - ./:/var/www  
    depends_on:  
      - web
```

#### Entendo a nossa anotação:

1.  Definimos nosso segundo serviço como **```nginx```**;
2.  Definimos onde busca a configuração da imagem no **Dockerfile** dentro de **```./.docker/nginx```**;
3.  Assim com os outros definimos que ele sempre irá reiniciar em (**```restart:always```**);
4.  Definimos o acesso pela nossa porta local **```:8000```** acessa a porta **```:80```** do container
5.  Definimos o [**volume**](https://docs.docker.com/storage/volumes/) de nossa aplicação, isto irá dizer para o docker ficar de olho em nossa pasta local, e ao mesmo tempo “**linkar**” com a pasta **```/var/www/```** do nosso container.
6.  E por fim adicionamos a dependência ao serviço **```web```**.

Agora que já temos quase tudo configurado, adicione o serviço de **```web```** a dependência do serviço de **```db```**.

```yml
  web:  
    restart: always  
    build: .  
    volumes:  
      - ./:/var/www/  
    depends_on:  
      - db
```
Seu arquivo [**```./docker-compose.yml```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/blob/master/docker-compose.yml) deve estar assim:

### Testando nossos containers

Para fazer um teste inicial no nosso terminal execute o comando para gerar o build dos nossos containers:
```sh
docker-compose up -d —build
```
Aguarde a finalização do processo.

> Se houver algum erro em seu build tente refazer os passos novamente, caso ocorra tudo bem você pode acessar via navegador o seu:
```
http://localhost:8000/
```
![Tela aplicação Laravel](https://cdn-images-1.medium.com/max/800/0*la8QHeZvfykrY-ID)

> Obs.: Caso de erro de permissão na pasta storage apenas rode na sua máquina local o comando: **chmod -R 777 ./storage** :v:

### Finalizando configuração

Legal, agora está funcionando como eu faço para me conectar ao banco e rodar minhas migrações?

Bem, primeiramente vamos configurar nossas variáveis de ambiente para o Laravel, podemos usar o arquivo **```.env```**.

Abra o arquivo [**```.env```**](https://github.com/VitorHugoSilva/exemple-laravel-with-docker/blob/master/.env) e faça as configurações para conectar ao seu banco.
```php
DB_CONNECTION=pgsql  
DB_HOST=db  
DB_PORT=5432  
DB_DATABASE=blog  
DB_USERNAME=postgres  
DB_PASSWORD=postgres
```
Pronto! ao configurar o **```DB_HOST```** para **```db```** automaticamente será feito o direcionamento para o serviço de **```db```**.

### Executado minhas migrações.

Você pode executar o comando direto em seu container para fazer a execução das migrações e demais operações:

Acessando container **```web```** via **```docker-compose exec```**.

No seu terminal execute o comando:

```sh
docker-compose exec web sh
```

Pronto agora você está dentro do container e pode executar suas ações normalmente como a migração.

![Comando docker-compose exec](https://cdn-images-1.medium.com/max/800/0*xcDY3qAFpMzoR11n)
```sh
php artisan migrate
```
![Executando migrations](https://cdn-images-1.medium.com/max/800/0*SwCSQRBbihd19lpt)

Para sair do container basta digitar
```sh
exit
```

### Parabéns você aprendeu uma das formas de se trabalhar com **Laravel** e **Docker**!!

### Dica:

Agora que você já sabe o básico, outras coisa que pode facilitar sua vida na hora de montar um ambiente de desenvolvimento php baseado em Docker é o [**Laradock**](https://laradock.io/), garanto que vai facilitar muito sua vida!

### Fontes e referências:

- [https://laravel.com/docs/6.x#installation](https://laravel.com/docs/6.x#installation)
-  [https://www.docker.com/resources/what-container](https://www.docker.com/resources/what-container)
- [https://laravel.com/docs/6.x/deployment#nginx](https://laravel.com/docs/6.x/deployment#nginx)
- [https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)
- [https://docs.docker.com/storage/volumes/](https://docs.docker.com/storage/volumes/)
- [https://yaml.org/](https://yaml.org/)
- [https://alpinelinux.org/](https://alpinelinux.org/)
