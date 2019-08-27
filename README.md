# Linux Server Configuration

#CARO REVISOR NÃO ENVIEI O CONTEÚDO DO SSH PELO COMENTÁRIO, MAS DEIXEI AQUI NO README. ESPERO QUE POSSA CONSIDERAR. OBRIGADO

### IP - 54.82.75.47
### Porta SSH - 2200
### URL - http://54.82.75.47.xip.io/

# Passo a Passo

## Criando uma instância Lightsail
- Enter https://lightsail.aws.amazon.com/ls/webapp/home/instances
- Plataform: Linux/Unix
- Blueprint: OS Only - Ubuntu 16.04 LTS
- Create Instance

## Antes de Conectar
- Em Networking adicione a configuração ao Firewall
    - Porta: 2200
    - Application: Custom
    - Protocol: TCP

## Criando par de chaves SSH no terminal local
Estou usando o terminal IOS
```sh
$ ssh-keygen
```
Resultará em:
```
Enter file in which to save the key (/Users/eneasmarques/.ssh/id_rsa): 
```
Informe o caminho e nome que deseja para as chaves.
```
./.ssh/my_key
```

## Conectar usando SSH
- Depois de logado iremos atualizar os pacotes
```
$ sudo apt-get update
$ sudo apt-get dist-upgrade 
```
*selecionar - install the package maintainer's version*
- Remover pacotes desnecessários
```
$ sudo apt-get autoremove
```

### Alterar a Porta SSH
- Editando arquivos no terminal
*CTRL+O(salva), ENTER(confirma), CTRL+X(sai)*
```
$ sudo nano /etc/ssh/sshd_config
```
*Alterar Port 22 para Port 2200*

- Reinicie serviço SSH 
```
$ sudo service ssh restart
```
### Permitir somente conexões de entrada para SSH (porta 2200), HTTP (porta 80) e NTP (porta 123)
- Verifique o status atual do UFW
```
$ sudo ufw status
```
- Bloquear todas as solicitações recebidas inicialmente
```
$ sudo ufw default deny incoming
```
- Permitir todas as conexões de saída
```
$ sudo ufw default allow outgoing
```
- Permitir porta de configuração SSH para 2200
```
$ sudo ufw allow 2200/tcp
```
- Permitir HTTP na porta padrão 80
```
$ sudo ufw allow www
```
- Permitir NTP em sua porta padrão 123
```
$ sudo ufw allow ntp
```
- Ativar firewall
```
$ sudo ufw enable
```
- Verifique os estados do UFW mais uma vez
```
$ sudo ufw status
```
## Criando e Configurando usuário grader
- Cria uma nova conta de usuário chamada grader
```
$ sudo adduser grader
```
- Permissão para sudo
```
$ sudo touch /etc/sudoers.d/grader
$ sudo ls -al /etc/sudoers.d
$ sudo nano /etc/sudoers.d/grader
```
*Adicione uma única linha contendo o **grader ALL = (ALL: ALL) ALL***

## Usando ssh-keygen
- Na terminal local copie o conteúdo da chave gerada anteriormente
```
$ cat ~/ssh/my_key.pub
```
- No servidor
```
$ sudo mkdir /home/grader/.ssh
$ sudo touch /home/grader/.ssh/authorized_keys
$ sudo nano /home/grader/.ssh/authorized_keys
```
*Cole o conteúdo de my_key.pub dentro do arquivo authorized_keys*

- Alterar o status de propriedade da pasta /.ssh
```
$ sudo chown grader:grader /home/grader/.ssh
```
- Alterar as credenciais da pasta /.ssh e authorized_keys
```
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 644 /home/grader/.ssh/authorized_keys
```
- Reinicie o serviço
```
$ sudo service ssh restart
```
- - Já é possível conectar no servidor pelo terminal local
```
$ ssh grader@PUBLIC-IP -p 2200 -i PATH-TO-MY-KEY-PUB-FILE
```
*No meu caso, PUBLIC-IP é 54.82.75.47 e PATH-TO-MY-KEY-FILE é ~/.ssh/my_key.pub*
## Remover acesso de usuário root e autenticação de senha
```
$ sudo nano /etc/ssh/sshd_config
```
*Altere PermitRootLogin para no; PasswordAuthenticationto no; No final do arquvivo acrescentar DenyUsers root*
## Configure o fuso horário local para UTC.
```
$ sudo timedatectl set-timezone UTC
```
## Instalar Apache e mod_wsgi para python 3
```
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi-py3 python3-dev
```
## Install Python 3 e pacotes essenciais
```
$ sudo apt-get install python3-pip
$ sudo python3 -m pip install --upgrade pip
$ sudo apt-get install libpq-dev python3-dev 
$ pip3 install psycopg2
```
## Instalar e configurar o POSTGRESQL
```
$ sudo apt-get install postgresql
$ sudo su - postgres

(postgres)$ psql
postgres=# CREATE DATABASE restaurantmenu;
postgres=# CREATE USER grader;
postgres=# ALTER ROLE grader WITH PASSWORD 'grader';
postgres=# GRANT ALL PRIVILEGES ON DATABASE restaurantmenu TO grader;
postgres=# \q
(postgres)$ exit
```
## Instalar o git e meu repositório de projetos Catálogo de Ítens
```
$ sudo apt-get install git
$ cd /var/www
$ sudo mkdir ItemCatalog
$ cd ItemCatalog
$ sudo git init
$ sudo git remote add origin https://github.com/eneasmarques/item-catalog-without-oauth.git
$ sudo git pull origin master
```
## Modificações realizadas no Projeto de Catálogo de Ítens
- Alteração do banco de dados para Postgres. Pode ser acompanhado no git
## Instale todas as bibliotecas no projeto de Catálogo de Ítens requirements.txt
```
$ sudo pip3 install -r requirements.txt
$ sudo python3 database_setup.py
$ sudo python3 database_populate.py
```
## Configurando o Apache
```
$ sudo nano /etc/apache2/sites-available/ItemCatalog.conf
```
*Copie o Código abaixo para o arquivo*
```
<VirtualHost *:80>
        ServerName 54.82.75.47
        ServerAdmin eneasmarqs@gmail.com
        ServerAlias 54.82.75.47.xip.io
	WSGIScriptAlias / /var/www/ItemCatalog/itemcatalog.wsgi
	<Directory /var/www/ItemCatalog>
		Order allow,deny
		Allow from all
	</Directory>
	Alias /static /var/www/ItemCatalog/static
	<Directory /var/www/ItemCatalog/static/>
		Order allow,deny
		Allow from all
	</Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	LogLevel warn
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Ativar host virtual e ativar novas configurações
```
$ sudo a2ensite ItemCatalog
$ sudo service apache2 reload
```
## Crie e configure o arquivo * .wsgi
```
$ sudo nano /var/www/ItemCatalog/itemcatalog.wsgi
```
*Adicione o código abaixo no arquivo*
```
#!/usr/bin/python3
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/ItemCatalog")

from __init__ import app as application
```
## Reinicie e recarregue o Apache
```
$ sudo service apache2 restart
$ sudo service apache2 reload
```
## Arquivo de Registros
```
$ sudo tail -f /var/log/apache2/error.log 
```
## Recursos de tereiros
### Courses

[Udacity's Linux Commandas Line Basics](https://www.udacity.com/course/viewer#!/c-ud595-nd)

[Udacity's Configuring Linux Web Servers](https://www.udacity.com/course/viewer#!/c-ud299-nd)

### Git
[@andrevst](https://github.com/andrevst/fsnd-p6-linux-server-configuration)

[@jungleBadger](https://github.com/jungleBadger/-nanodegree-linux-server)

[@bcko](https://github.com/bcko/Ud-FS-LinuxServerConfig-LightSail)

[@leandrocl2005](https://github.com/leandrocl2005/aws_lightsail_config_for_flask_python3)

