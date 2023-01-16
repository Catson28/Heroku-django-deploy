
Neste guia do Django Heroku, falarei sobre como implantar o projeto Django no Heroku usando o Docker.

Assim, após a leitura, você obterá:

1.  A diferença entre Heroku Buildpacks e Heroku Container.
2.  Como servir recursos estáticos e arquivos de mídia do Django no Heroku.
3.  Como testar a imagem do Docker para o projeto Django no ambiente local.
4.  Como construir material de front-end para o projeto Django durante a implantação.

# Topicos

### 01 - Heroku Buildpacks e Heroku Container
### 02 - Construir manifesto e Container Registry
### 03 - Docker Build x Docker Run
### 04 - Ambiente
### 05 - arquivo .dockerignore
### 06 - Comece a escrever Dockerfile
### 07 - Servindo recursos estáticos
### 08 - Criar imagem do Docker
### 09 - execução do Docker
### 10 - Servindo arquivos de mídia no Heroku
### 11 - Suporte a banco de dados remoto
### 12 - heroku.yml

## Implantar o projeto Django no Heroku
### 01 - Adicionar complemento de banco de dados
### 02 - Adicionar AWS ENV
### 03 - solucionar problemas
### 04 - Conclusão


O código-fonte deste tutorial é [django-heroku](https://github.com/AccordBox/django-heroku) . Eu apreciaria se você pudesse dar uma estrela.

## 01 - Heroku Buildpacks e Heroku Container

Existem basicamente duas maneiras de implantar o projeto Django no Heroku

Uma maneira é o `buildpacks`.

> Os buildpacks são responsáveis ​​por transformar o código implantado em um slug, que pode ser executado em um dinamômetro

Você pode ver `Buildpacks`alguns `pre-defined`scripts mantidos pela equipe Heroku que implementam seu projeto Django. Eles geralmente dependem de suas linguagens de programação.

`Buildpacks`é muito fácil de usar, então a maioria dos tutoriais do Heroku gostaria de falar sobre isso.

Outra forma é usando `Docker`.

O Docker nos fornece uma maneira mais flexível para que possamos ter mais controle, você pode instalar quaisquer pacotes que desejar no sistema operacional ou também executar quaisquer comandos durante o processo de implantação.

Por exemplo, se o seu projeto Django tiver um aplicativo front-end que ajude a compilar SCSS e ES6, você deseja `npm ci`alguns pacotes de dependência durante o processo de implantação e, em seguida, executar o `npm build`comando, o Docker parece uma solução mais limpa enquanto `buildpacks`pode fazer isso.

Além disso, `Docker`permite implantar o projeto de uma maneira que a maioria das plataformas pode suportar, pode economizar tempo se você quiser migrá-lo para outras plataformas, como AWS, Azure no futuro.

## 02 - Construir manifesto e Container Registry

Algumas pessoas estão confusas sobre `Build manifest`e `Container Registry`no documento Heroku Docker. Então deixe-me explicar aqui.

`Container Registry`significa que você cria o docker no local e, em seguida, envia a imagem para o Heroku Container Registry. O Heroku usaria a imagem para criar um contêiner para hospedar seu projeto Django.

`Build manifest`significa que você envia o Dockerfile e o Heorku o constrói e executa no fluxo de lançamento padrão.

Além disso, oferece `Build manifest`suporte a alguns recursos integrados úteis do Heorku, como `Review Apps`, `Release`.

Se você não tiver nenhum motivo especial, **recomendo fortemente o uso `Build Manifest`de uma maneira de implantar o Django.**

## 03 - Docker Build x Docker Run

Então, qual é a diferença entre `Docker build`e`Docker run`

`Docker build`cria imagens do Docker a partir de um Dockerfile.

> A `Dockerfile`é um documento de texto que contém todos os comandos que um usuário pode chamar na linha de comando para construir uma imagem.

`Docker run`crie uma camada de contêiner gravável sobre a imagem especificada,

Portanto, devemos primeiro usar `docker build`para criar a imagem do docker `Dockerfile`e, em seguida, criar um contêiner sobre a imagem.

## 04 - Ambiente

Primeiro, vamos adicionar `django-environ`a `requirements.txt`.

```ini
django-environ==0.8.1

```

E então, vamos atualizar as configurações do Django `django_heroku/settings.py`para usar `django-environ`para ler o env.

```python
import environ                           # new
from pathlib import Path

env = environ.Env()                      # new


# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env('DJANGO_SECRET_KEY', default='django-insecure-$lko+#jpt#ehi5=ms9(6s%&6fsg%r2ag2xu_2zj1ibsj$pckud')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = env.bool("DJANGO_DEBUG", True)

ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS", default=[])

```

Notas:

1.  Aqui usamos `django-environ`para definir `SECRET_KEY`, `DEBUG`e `ALLOWED_HOSTS`da variável Environment.
2.  E também definimos `default`valor para fazê-lo funcionar no desenvolvimento sem Env extra.

```bash
$ ./manage.py migrate
$ ./manage.py runserver

```

Agora, verifique em [http://127.0.0.1:8000/](http://127.0.0.1:8000/) para ter certeza de que tudo está funcionando.

## 05 - arquivo .dockerignore

> Antes que a CLI do docker envie o contexto para o daemon docker, ela procura um arquivo chamado .dockerignore no diretório raiz do contexto. Se esse arquivo existir, a CLI modificará o contexto para **excluir arquivos e diretórios que correspondam aos padrões nele** . Isso ajuda a evitar o envio desnecessário de arquivos e diretórios grandes ou confidenciais para o daemon e possivelmente adicioná-los às imagens usando ADD ou COPY

vamos criar`.dockerignore`

```perl
**/node_modules
**/build

```

Às vezes, se o seu projeto Django estiver usando NPM como solução de front-end, NÃO devemos adicionar arquivos `node_modules`ao contêiner do Docker.

Podemos usar `.dockerignore`, para que os arquivos `node_modules`e `build`o diretório NÃO sejam processados ​​pelo Docker quando `ADD`ou`COPY`

## 06 - Comece a escrever Dockerfile

Agora que você já tem um entendimento básico da janela de encaixe do Heroku, vamos aprender mais sobre `Dockerfile`,

Antes de começarmos, vamos dar uma olhada nas [compilações de vários estágios do docker](https://docs.docker.com/develop/develop-images/multistage-build/#use-multi-stage-builds)

> Com compilações de vários estágios, você usa várias instruções FROM em seu Dockerfile. Cada instrução FROM pode usar uma base diferente, e cada uma delas inicia uma nova etapa da construção. Você pode copiar seletivamente artefatos de um estágio para outro, deixando para trás tudo o que não deseja na imagem final.

Aqui eu usaria `django_heroku`como exemplo para mostrar como escrever Dockerfile.

```perl
# Please remember to rename django_heroku to your project directory name
FROM node:14-stretch-slim as frontend-builder

WORKDIR /app/frontend

COPY ./frontend/package.json /app/frontend
COPY ./frontend/package-lock.json /app/frontend

ENV PATH ./node_modules/.bin/:$PATH

RUN npm ci

COPY ./frontend .

RUN npm run build

#################################################################################
FROM python:3.10-slim-buster

WORKDIR /app

ENV PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app \
    DJANGO_SETTINGS_MODULE=django_heroku.settings \
    PORT=8000 \
    WEB_CONCURRENCY=3

# Install system packages required by Wagtail and Django.
RUN apt-get update --yes --quiet && apt-get install --yes --quiet --no-install-recommends \
    build-essential curl \
    libpq-dev \
    libmariadbclient-dev \
    libjpeg62-turbo-dev \
    zlib1g-dev \
    libwebp-dev \
 && rm -rf /var/lib/apt/lists/*

RUN addgroup --system django \
    && adduser --system --ingroup django django

# Requirements are installed here to ensure they will be cached.
COPY ./requirements.txt /requirements.txt
RUN pip install -r /requirements.txt

# Copy project code
COPY . .
COPY --from=frontend-builder /app/frontend/build /app/frontend/build

RUN python manage.py collectstatic --noinput --clear

# Run as non-root user
RUN chown -R django:django /app
USER django

# Run application
CMD gunicorn django_heroku.wsgi:application

```

1.  Para frontend, o que queremos são os arquivos em `frontned/build`, não os pacotes em `node_modules`,
2.  Para o primeiro estágio de construção, atribuímos um nome`frontend-builder`
3.  No primeiro estágio de compilação, instalamos pacotes de dependência de front-end e usamos `npm run build`para construir ativos estáticos, após a conclusão deste comando, os ativos construídos estariam disponíveis em`/app/frontend/build`
4.  No segundo estágio de compilação, após `COPY . .`, copiamos `/app/frontend/build`do primeiro estágio e colocamos em`/app/frontend/build`
5.  `--from`tem o valor de `build stage`, aqui o valor é `frontend-builder`, que definimos em`FROM node:12-stretch-slim as frontend-builder`
6.  Observe que a `WORKDIR`instrução docker, ela define o diretório de trabalho para as instruções docker (RUN, COPY, etc.) É muito parecido com o `cd`comando no shell, mas você não deve usar `cd`no Dockerfile.
7.  Como você já sabe o que é `docker run`and `docker build`, quero dizer que quase todas as instruções do docker acima seriam executadas em `docker build`. A única exceção é a última `CMD`, seria executada em `docker run`.
8.  Usamos `ENV`para definir a variável env padrão.
9.  Adicionamos um `django`usuário e o usamos para executar o comando por segurança.

Adicionar `gunicorn`a _requirements.txt_

```ini
gunicorn==20.1.0

```

## 07 - Servindo recursos estáticos

O Django serve apenas arquivos de mídia e recursos estáticos no modo dev, então mostrarei como servi-los no Heroku no modo de produção.

Para veicular recursos estáticos, precisamos usar um pacote de terceiros. `whitenoise`.

Adicionar `whitenoise`a _requirements.txt_

```ini
whitenoise==5.3.0

```

Atualizar`django_heroku/settings.py`

```python
import os                                                                            # new

MIDDLEWARE = [
   'django.middleware.security.SecurityMiddleware',
   'whitenoise.middleware.WhiteNoiseMiddleware',                                     # new
   # ...
]

STATIC_URL = 'static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static')                                       # new

STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'frontend/build'),                                        # you can change it
]

STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"      # new

```

Essa é toda a configuração que você precisa.

1.  Depois `python manage.py collectstatic --noinput --clear`no Dockerfile executado em `docker build`, todos os ativos estáticos seriam colocados no `static`diretório.
2.  Depois `docker run`de executado no Heroku, o Django pode servir ativos estáticos sem problemas.

## 08 - Criar imagem do Docker

Depois de criá -lo `Dockerfile`e colocá-lo na raiz do seu projeto Django, é recomendável testá-lo localmente, isso pode economizar seu tempo se algo estiver errado.

PS: Se você não instalou o Docker, verifique este documento de [instalação do Docker](https://docs.docker.com/install/)

```bash
$ docker build -t django_heroku:latest .

```

Em alguns casos, você também pode querer não usar o`build cache`

Você pode usar a `--no-cache=true`opção e verificar [aproveitar-construir-cache](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache) para saber mais.

## 09 - execução do Docker

Se a imagem do docker foi construída sem erros, aqui podemos continuar verificando no ambiente local

```bash
$ docker run -d --name django-heroku-example -e "PORT=9000" -e "DJANGO_DEBUG=0" -e "DJANGO_ALLOWED_HOSTS=*" -p 9000:9000 django_heroku:latest

# Now visits http://127.0.0.1:9000/admin/

```

1.  Em `Dockerfile`, usamos `ENV`para definir a variável env padrão em `docker run`, mas ainda podemos usar `-e "PORT=9000"`para substituir o env no comando run.
2.  `-p 9000:9000`deixe-nos visitar a porta 9000 no contêiner até a 9000 na máquina host.
3.  Vamos fazer algumas verificações e os scripts abaixo também podem ajudá-lo a solucionar problemas.

```shell
$ docker exec -it django-heroku-example bash

django@709bf089caf0:/app$ ./manage.py shell
Python 3.10.1 (main, Dec 21 2021, 09:50:13) [GCC 8.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>> from django.conf import settings
>>> settings.DEBUG
False

```

```bash
# cleanup
$ docker stop django-heroku-example
$ docker rm django-heroku-example

```

## 10 - Servindo arquivos de mídia no Heroku

Como todas as alterações no contêiner docker seriam perdidas durante a reimplantação. Portanto, precisamos armazenar nossos arquivos de mídia em outro local, em vez do contêiner do Heroku Docker.

A solução popular é usar o armazenamento Amazon S3, porque o serviço é muito estável e fácil de usar.

1.  Se você não tiver uma conta de serviço da Amazon, acesse [Amazon S3](https://aws.amazon.com/s3/) e clique em `Get started with Amazon S3`para se inscrever.
2.  Faça login [no Console de gerenciamento da AWS](https://console.aws.amazon.com/console/home)
3.  No canto superior direito, clique no nome da sua empresa e, em seguida, clique em`My Security Credentials`
4.  Clique na `Access Keys`seção
5.  `Create New Access Key`, copie o `AMAZON_S3_KEY`e `AMAZON_S3_SECRET`para o notebook.

Em seguida, começamos a criar o Amazon bucket no [console de gerenciamento S3](https://console.aws.amazon.com/s3/home) , copie `Bucket name`para o notebook.

`Bucket`no Amazon S3 é como um contêiner de nível superior, cada site deve ter seu próprio `bucket`, e o nome do bucket é exclusivo em todo o Amazon s3, e o URL dos arquivos de mídia tem domínio como `{bucket_name}.s3.amazonaws.com`.

Agora vamos configurar o projeto Django para deixá-lo usar o Amazon s3 no Heroku.

atualização `requirements.txt`.

```ini
boto3==1.16.56
django-storages==1.11.1

```

Adicionar `storages`a `INSTALLED_APPS`em`django_heroku/settings.py`

E então atualize`django_heroku/settings.py`

```python
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')                                      # new
MEDIA_URL = '/media/'                                                             # new

if 'AWS_STORAGE_BUCKET_NAME' in env:                                              # new
    AWS_STORAGE_BUCKET_NAME = env('AWS_STORAGE_BUCKET_NAME')
    AWS_S3_CUSTOM_DOMAIN = '%s.s3.amazonaws.com' % AWS_STORAGE_BUCKET_NAME
    AWS_ACCESS_KEY_ID = env('AWS_ACCESS_KEY_ID')
    AWS_SECRET_ACCESS_KEY = env('AWS_SECRET_ACCESS_KEY')
    AWS_S3_REGION_NAME = env('AWS_S3_REGION_NAME')
    AWS_DEFAULT_ACL = 'public-read'
    AWS_S3_FILE_OVERWRITE = False

    MEDIA_URL = "https://%s/" % AWS_S3_CUSTOM_DOMAIN
    DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'

```

1.  Para proteger seu projeto Django, defina `AWS_STORAGE_BUCKET_NAME`, `AWS_ACCESS_KEY_ID`  `AWS_SECRET_ACCESS_KEY`e `AWS_S3_REGION_NAME`em ENV em vez do código-fonte do projeto (mostrarei como fazer isso no Heroku daqui a pouco)
2.  **`AWS_S3_FILE_OVERWRITE`por favor, defina-o como False** , para que isso possa permitir que o armazenamento manipule o `duplicate filenames`problema. (Não entendo porque tantos posts de blog não mencionam isso)

Observe que `AWS_ACCESS_KEY_ID`e `AWS_SECRET_ACCESS_KEY`é para AWS `admin`, para melhor segurança, você deve criar `IAM`um usuário AWS e, em seguida, conceder permissões S3. Você pode verificar [o proprietário do bucket concedendo permissões de bucket aos seus usuários](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-walkthroughs-managing-access-example1.html) para saber mais.

## 11 - Suporte a banco de dados remoto

O Heroku suporta muitos bancos de dados, você pode escolher o que quiser e adicioná-lo à sua instância do Heroku. (recomendo `PostgreSQL`)

No Heroku, a string de conexão do banco de dados é anexada como variável ENV. Assim, podemos definir nossas configurações dessa maneira.

Atualizar`django_heroku/settings.py`

```python
if "DATABASE_URL" in env:
    DATABASES['default'] = env.db('DATABASE_URL')
    DATABASES["default"]["ATOMIC_REQUESTS"] = True

```

Aqui `env.db`seria convertido `DATABASE_URL`para Django db connection dict para nós.

Não se esqueça de adicionar `psycopg2-binary`a `requirements.txt`.

```ini
psycopg2-binary==2.9.2

```

## 12 - heroku.yml

`heroku.yml`é um manifesto que você pode usar para definir seu aplicativo Heroku.

Por favor, crie um arquivo na raiz do diretório

```yaml
build:
  docker:
    web: Dockerfile
release:
  image: web
  command:
    - django-admin migrate --noinput

```

1.  Como você pode ver, no `build`estágio, o docker criaria a `web`imagem a partir do arquivo `Dockerfile`.
2.  No `release`estágio, `migrate`o comando seria executado para nos ajudar a sincronizar nosso banco de dados.

Do [documento Heroku](https://devcenter.heroku.com/articles/container-registry-and-runtime)

> Se você deseja ver os logs de streaming conforme a fase de lançamento é executada, sua imagem do Docker deve ter `curl`. Se a imagem do Docker não incluir o curl, os logs da fase de lançamento estarão disponíveis apenas nos logs do aplicativo.

Já instalamos `curl`em nosso`Dockerfile`

# Implantar o projeto Django no Heroku

Agora, vamos começar a implantar nosso projeto Django no Heorku.

Primeiro, vamos ao site Heroku para fazer login e criar um aplicativo Heroku. ( `django-heroku-docker`neste caso)

Depois de criar o aplicativo, podemos obter o comando shell que pode nos ajudar a implantar o projeto no Heroku.

![](https://www.accordbox.com/upload/images/django-heroku-docker-deploy-setting.original.png)

Então começamos a configurar e implantar no terminal.

```bash
$ heroku login
$ heroku git:remote -a django-heroku-docker
$ heroku stack:set container -a django-heroku-docker

# git add files and commit, you can change to main branch
$ git push heroku master:master

```

1.  `heroku stack:set container`é importante aqui porque diria ao Heroku para usar o contêiner em vez de `buildpacks`implantar o projeto.
2.  Você pode encontrar o domínio do seu aplicativo Heroku na `settings`guia. (Heroku tem plano gratuito para você testar e aprender como quiser)

Depois que o código for implantado no Heroku, não se esqueça de adicionar ENV `DJANGO_ALLOWED_HOSTS`e `DJANGO_DEBUG`ao `DJANGO_SECRET_KEY`aplicativo Heroku.

Em seguida, verifique [https://django-heroku-docker.herokuapp.com/admin/](https://django-heroku-docker.herokuapp.com/admin/) para ver se tudo funciona.

## 01 - Adicionar complemento de banco de dados

Agora você pode adicionar o complemento db à sua instância do Heroku, para que os dados do seu projeto Django sejam persistentes.

1.  Vá para a `overview`guia do seu projeto Heroku, clique em`Configure Add-ons`
2.  Pesquise `Heroku Postgres`e clique `Provision`no botão.
3.  Agora vá para a `settings`aba do seu projeto Heroku, clique no botão`Reveal Config Vars`
4.  Você verá `DATABASE_URL`, é a variável ENV do seu aplicativo Heroku

```bash
# let's check
$ heroku config -a django-heroku-docker

DATABASE_URL:         postgres://xxxxxxx

```

Não se esqueça de verificar o Heroku `Release log`para ver se a migração foi executada:

```yaml
Running migrations:
  No migrations to apply.

```

## 02 - Adicionar AWS ENV

Agora você pode adicionar Env `AWS_STORAGE_BUCKET_NAME`, `AWS_ACCESS_KEY_ID`e no Heroku `AWS_SECRET_ACCESS_KEY`e `AWS_S3_REGION_NAME`verificar o recurso de upload de mídia.

## 03 - solucionar problemas

Se você encontrar um problema, poderá usar o log do Heroku para verificar mais detalhes.

## 04 - Conclusão

Neste tutorial do Django Heorku, falei sobre como implantar o projeto Django no Heroku usando o Docker.

Você pode encontrar o código-fonte [django-heroku](https://github.com/AccordBox/django-heroku) . Eu apreciaria se você pudesse dar uma estrela.

> Série de tutoriais Django Heroku:
> 
> 1.  [Heroku vs AWS Qual é o melhor para o seu projeto Django](https://www.accordbox.com/blog/heroku-vs-aws-which-best-your-django-project/)
> 2.  [Como implantar o projeto Django no Heroku usando o Docker](https://www.accordbox.com/blog/deploy-django-project-heroku-using-docker/)
> 3.  [Como implantar o projeto Python no Heroku no Gitlab CI](https://www.accordbox.com/blog/how-deploy-python-project-heroku-gitlab-ci/)
> 4.  [Como usar o pipeline Heroku](https://www.accordbox.com/blog/how-use-heroku-pipeline/)
> 5.  [Tutorial de logs do Heroku](https://www.accordbox.com/blog/heroku-logs-tutorial/)
> 6.  [Como monitorar o Heroku Postgres usando heroku-pg-extras](https://www.accordbox.com/blog/how-monitor-postgres-using-heroku-pg-extras/)
> 
> Série de tutoriais Django Dokku:
> 
> 1.  [Como implantar o projeto Django no Dokku](https://www.accordbox.com/blog/how-deploy-django-project-dokku/)
> 2.  [Como implantar o projeto Django no Dokku com o Docker](https://www.accordbox.com/blog/how-deploy-django-project-dokku-docker/)

Lançar produtos mais rapidamentecom Django

O SaaS Hammer ajuda você a lançar produtos de maneira mais rápida. Ele contém todas as bases que você precisa para que você possa se concentrar em seu produto.

[Saber mais](https://saashammer.com/)

![](https://www.accordbox.com/upload/images/4366781.original.jpg)

Michael Yin

Michael é um desenvolvedor Full Stack da China que adora escrever código, tutoriais sobre Django e tecnologia de front-end moderna.

Ele publicou alguns ebooks em [leanpub](https://leanpub.com/u/michaelyin) e cursos de tecnologia em [testdriven.io](https://testdriven.io/authors/yin/) .

Ele também é o fundador da AccordBox, que fornece serviços de desenvolvimento web.