# GitHub Actions + PostgreSQL + RoR no Oracle Cloud

## Introdução

O procedimento apresentado nesta documentação descreve os passos para o instanciar uma (ou mais) Virtual Machine (VM) dentro do Oracle Cloud para execução de soluções baseadas em Ruby on Rails (RoR) com PostgreSQL. O provisionamento e configuração da aplicação será disparado desde o pipeline do GitHub Actions.

O procedimento está dividido em 5 etapas:

- Pré-Requisitos
- PostgreSQL no OCI em HA utilizando o Terraform (Oracle Resource Manager)
- Criação do script de WorkLoad no GitHub Actions
- Inicialização da Aplicação
- Criação do Load Balancer no Oracle Cloud


## 1. Pré-Requisitos

### ✋ Conta, APIKey e Subnet no Oracle Cloud

Para criar Config File, ingresse ao Oracle Cloud, vá a Identidadee/Segurnaça >  Usuário, selecione o usuário que terá acesso, crie uma API KEY> Solicite para que o public e private PEM sejam gerados junto ao processo.

1. Copie o conteudo do config file e do pem file (será utilizado na configuração no GitHub)

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/api_key.png"/></th>
        </tr>
    </tbody>
</table>

📎 importante ter uma subnet criada pois ela não será provisionada. Deverá ter habilitado o ingress rules para portas 80, 8080 e 3000.


### ✋ Conta e segredos no projeto do GitHub

Nos seguintes passos será criado o script oci.yml dentro do GitHub Action.

Esse script necessitará de nove segredos inicialmente: **CONFIG**, **OCI_KEY_FILE**, **ID_RSA_PRIV** e **ID_RSA**. Outros 4 segredos são: **GIT_USER** (nome do usuario git), **GIT_SECRET** (token gerado no git) e **GIT_CLONE_URL** (path do projeto que queremos fazer o pull, exemplo: github.com/fcostabr78/cicd.git). O ultimo segredo é o **OCI_COMPARTMENT_ID** e **OCI_SUBNET_ID** que deverão ter o OCI_ID correspondente a localização que será realizada o provisionamento dos servidores de aplicação.

```
echo "${{secrets.CONFIG}}" >> ~/.oci/config
echo "${{secrets.OCI_KEY_FILE}}" >> ~/.oci/key.pem
echo "${{secrets.ID_RSA}}" >> ~/.oci/id_rsa.pub
```
1. Para gerar o secret no GitHub :octocat:, em seu projeto vá a Setting, depois Secrets. 
2. Clique no botão "New Repository Secret" para criar as secrets descritas acima.

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/action_secret_git2.png"/></th>
        </tr>
    </tbody>
</table>

⚠️ **os valores dos segredos CONFIG e OCI_KEY_FILE foram obtidos no passo anterior, via criação da API KEY na console do Oracle Cloud**<br>
⚠️ o valor do segredo **ID_RSA** pode ser obtido localmente com *$ cat /home/<user>/.ssh/id_rsa.pub*<br>
⚠️ o valor do segredo **ID_RSA_PRIV** pode ser obtido localmente com *$cat /home/fernando/.ssh/id_rsa*<br>
    
Fique atento ao path de key_file que deverá ser informado no valor do segredo do CONFIG. O arquivo gerado na criação da API KEY anterior terá o seguinte formato:

```
[DEFAULT]
user=ocid1.user.oc1..aaaaaaaa3wqzj7nxxfkrnz7y4ewt.....
fingerprint=51:4f:c2:79:b3:78:7a:48:89:2f:01:97:11:01:f4:e0
tenancy=ocid1.tenancy.oc1..aaaaaaaaqqzek25x6.....
region=us-ashburn-1
key_file=~/.oci/key.pem
```
    
3. Criar um token em https://github.com/settings/tokens para permitir clonar projeto privado. Copie o token gerado e salve num local seguro. Uma sugestão ao futuro é criar uma secret e adicionar esse valor do token lá.

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/token.png"/></th>
        </tr>
    </tbody>
</table>
    

## 2. PostgreSQL no OCI em HA

1. Via navegador acesse https://docs.oracle.com/pls/topic/lookup?ctx=pt-br/solutions/deploy-postgresql-db&id=github-oci-postgresql-stack-zip
2. Informe sua credenciais de conta ao Oracle Cloud
3. Clique no checkbox para aceitar os termos e condições
4. Selecione o compartiment onde deseja provisonar os recursos de DB
5. Clique em "Next"
6. Selecione um Availability Domain ao master PostgreSQL
7. Ingresse uma senha padrão em "PostgreSQL Password"
8. Para efeito de teste colocaremos o banco numa subnet publica (mas o ideal é na subnet privada)
9. Clique em "Show advanced options"
10. Cole sua SSH Key
11. Clique em "Next"
12. Apply 

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/pg.png"/></th>
        </tr>
    </tbody>
</table>
    
    
### Alterar Senha e Liberar Acesso no Postgres

```
$ ssh -i /home/fernando/.ssh/id_rsa opc@<IP_MASTER_PG>
$ sudo su postgres
$ psql -c "ALTER USER postgres WITH PASSWORD '<novasenha>'"

$ vim /var/lib/pgsql/13/data/postgresql.conf
  - retirar o comentário (#) do parametro listen_address
  - alterar o valor da propriedade de listen_address de [localhost] para *
  - retirar o comentário (#) do parametro port

$ vim /var/lib/pgsql/13/data/pg_hba.conf
adicionar as linhas:
- host all all 0.0.0.0/0 password
- host all all ::0/0 password

$ exit

$ sudo firewall-cmd --permanent --zone=trusted --add-source=0.0.0.0/0 
$ sudo firewall-cmd --permanent --zone=trusted --add-port=5432/tcp
$ sudo firewall-cmd --reload

Reiniciar o master node do PG

psql -h <IP_MASTER_PG> -U postgres
```
    
<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/pg_a.png"/></th>
        </tr>
    </tbody>
</table>

**Para instalar o PostgreSQl Client no Ubuntu: $sudo apt install postgresql-client**
    
Crie as seguintes secrets no GitHub: **DB_USER**, **DB_PASSWORD**, **DB_HOST** e **DB_PORT**. O password a ser informado deve ser o registrado no passo anterior e o host é o IP publico do master node provisionado ao PostgreSQL. 
    
## 3. Criar WorkLoad no GitHub Actions

1. Dentro do projeto GitHub, clique em <b>"Actions"</b>, depois no botão <b>"New Workload"</b>

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/workload.png"/></th>
        </tr>
    </tbody>
</table>

2. Ao editar o arquivo blank.yml, o renomeie para <b>oci.yml</b> e substitua o conteúdo por este abaixo

> :warning: Em seu projeto você poderá ter um mais workload<br>
> :warning: Cada workload determinará **sua condição de execução**. Para isso verifique a condição deterimada em **on** e os eventos. No exemplo abaixo o script será executado nos eventos **push** e **pull_request**<br>
> :warning: **O script liberará no firewall as portas necessárias ao rails**<br>
> :warning: No script abaixo de exemplo, o comando **oci compute instance launch** cria uma VM de tipo *VM.Standard.E2.1* e a imagem foi atribuida no parâmetro --image-id    
 
``` yml
# This is a basic workflow to help you get started with Actions

name: CI

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    outputs:
      foo1-output: ${{ steps.generate-output.outputs.bar-output }}
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2

      # Runs a single command using the runners shell
      - name: Inicia script 
        run: echo Ola Oracle!

      - name: 'Cria o arquivo de configuracao com base no OCI'
        run: |
          mkdir ~/.oci
          echo "${{secrets.CONFIG}}" >> ~/.oci/config
          echo "${{secrets.OCI_KEY_FILE}}" >> ~/.oci/key.pem
          echo "${{secrets.ID_RSA}}" >> ~/.oci/id_rsa.pub

      - name: 'Instalar OCI CLI'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        run: |
          curl -L -O https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh
          chmod +x install.sh
          ./install.sh --accept-all-defaults
          echo "::add-path::/home/runner/bin"
          exec -l $SHELL

      - name: 'Corrigir permissoes de arquivos criados'
        run: |
          oci setup repair-file-permissions --file /home/runner/.oci/config
          oci setup repair-file-permissions --file /home/runner/.oci/key.pem

      - name: 'Criar uma instancia'
        id: instancia
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        run: |
          INSTANCE=$(oci compute instance launch --ssh-authorized-keys-file ~/.oci/id_rsa.pub --availability-domain "AHhM:US-ASHBURN-AD-1" --compartment-id ${{secrets.OCI_COMPARTMENT_ID}} --shape "VM.Standard.E2.1" --display-name "app_server" --image-id ocid1.image.oc1.iad.aaaaaaaatwjeakck3drug6mmutcz3msodjse56qxdtwnvehldu7yds66r2wq --subnet-id ${{secrets.OCI_SUBNET_ID}} --wait-for-state RUNNING)
          INSTANCE_ID=$(echo $INSTANCE | jq -r '.data.id')
          echo $INSTANCE_ID

          IP=$(oci compute instance list-vnics --instance-id $INSTANCE_ID --query 'data [0]."public-ip"' --raw-output)
          echo "::set-output name=IP::$(echo "$IP")"

      - name: Check IP
        run: |
          echo "O IP criado com sucesso foi ${{ steps.instancia.outputs.IP }}"

      - name: 'Validar o acesso via SSH'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        run: |
          while ! nc -w5 -z ${{ steps.instancia.outputs.IP }} 22; do
                  sleep 5
                  echo "SSH not available..."
          done; echo "SSH ready!"
      
      - name: 'Testar Comando SSH'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          script: whoami
      
      - name: 'Atualizar pacotes (Ruby, RubyGem, OCI Utils, Ansible, Terraform, Java...) e instalar o bundler'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          command_timeout: 700m
          script: |
           sudo yum update -y
           gem install bundler

      - name: 'Instalar PowerTools e ImageMagick'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          command_timeout: 300m
          script: |
           sudo dnf install -y epel-release
           sudo dnf config-manager --set-enabled PowerTools
           sudo dnf install -y ImageMagick ImageMagick-devel

      - name: 'Instalar Dependencias, SQLite, MySQL-Devel, Postgresql lib'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          script: |
           yes Y | sudo dnf install sqlite-devel sqlite-libs mysql-server mysql-devel postgresql-server postgresql-devel redis memcached libxml2-devel
           
      - name: 'Instalar Yarn'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          command_timeout: 300m
          script: |
           curl --silent --location https://rpm.nodesource.com/setup_8.x | sudo bash -
           curl --silent --location https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
           yes Y | sudo dnf install yarn -y
 
      - name: 'Instalar Ruby Devel'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          command_timeout: 300m
          script: |
           yes Y | sudo yum install ruby-devel
           
      - name: 'Instalar Rails 6.1.0'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          command_timeout: 300m
          script: |
           gem install rails -v 6.1.0
           rails -v

      - name: 'Liberacao de Acessos'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          command_timeout: 300m
          script: |
           sudo firewall-cmd --permanent --add-port=80/tcp
           sudo firewall-cmd --permanent --add-port=8080/tcp
           sudo firewall-cmd --permanent --add-port=3000/tcp
           sudo firewall-cmd --add-service=http --permanent
           sudo firewall-cmd --reload
           
      - name: 'Deploy do projeto'
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: 'true'
        uses: appleboy/ssh-action@master
        with:
          host: ${{ steps.instancia.outputs.IP }}
          username: opc
          key: ${{ secrets.ID_RSA_PRIV}} 
          port: 22
          command_timeout: 300m
          script: |
           git config --global user.name "Fernando"
           git config --global user.email fdacosta1978@gmail.com
           git clone -b master https://${{ secrets.GIT_USER}}:${{ secrets.GIT_SECRET}}@${{ secrets.GIT_CLONE_URL}} agenda
           cd agenda
           bundle install
           sudo gem pristine --all
           cat <<EOF > config/database.yml
           production:
             adapter: postgresql
             encoding: unicode
             database: oradev_production
             pool: 5
             username: ${{ secrets.DB_USER}}
             password: ${{ secrets.DB_PASSWORD}} 
             host: ${{ secrets.DB_HOST}}
             port: ${{ secrets.DB_PORT}}
           development:
             adapter: postgresql
             encoding: unicode
             database: oradev_development
             pool: 5
             username: ${{ secrets.DB_USER}}
             password: ${{ secrets.DB_PASSWORD}} 
             host: ${{ secrets.DB_HOST}}
             port: ${{ secrets.DB_PORT}}
           test:
             adapter: postgresql
             encoding: unicode
             database: oradev_test
             pool: 5
             username: ${{ secrets.DB_USER}}
             password: ${{ secrets.DB_PASSWORD}} 
             host: ${{ secrets.DB_HOST}}
             port: ${{ secrets.DB_PORT}}
           EOF
           rails db:setup
           rails db:migrate
           rails webpacker:install
```


> :partying_face: Após a execução do script acima, a VM com RoR estará provisionado no OCI, com nome "app_server", assim como o deploy estará realizado e o Actions deverá informar que a execução foi exitosa.

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/final.png"/></th>
        </tr>
    </tbody>
</table>

## 4. Inicializar a aplicação
    
Será possível acessar a VM desde a que tenham as chaves e o arquivo de configuração informado.

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/access_local.png"/></th>
        </tr>
    </tbody>
</table>

Dentro da pasta do  projeto digite *rails server -b 0.0.0.0 -p 3000*
    
Na console do Oracle Cloud, a VM instanciada apresenta o IP Publico atribuído. Basta chamar pelo navegador na porta 3000:
    
<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/rails.png"/></th>
        </tr>
    </tbody>
</table>
    
 
## 5. Criar Load Balancer à aplicação
    
1. Desde a console do OCI selecione Networking > Load Balancers
2. Clique no botão **"Create Load Balancer"**
3. Selecione "Load Balancer" e depois no botão "Create"
<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/lb1.png"/></th>
        </tr>
    </tbody>
</table>
4. Atribua um load balancer público de 100 mbps com IP efemero. Atenção para ele estar **na mesma VCN e subnet que a VM de aplicação foi criada**.
<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/lb2.png"/></th>
        </tr>
    </tbody>
</table>
5. Determine a distribuição Round Robin, selecione de backend set a VM criada no passo anterior. **Atenção a porta que deve ser 3000**.
<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/lb3.png"/></th>
        </tr>
    </tbody>
</table>
<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/lb31.png"/></th>
        </tr>
    </tbody>
</table>

Realizado os passos anteriores o LB será criado e apresentado conforme abaixo
    
<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/lb4.png"/></th>
        </tr>
    </tbody>
</table>

Copiar o endereço do IP criado no passo anterior e fazer o **request sem necessidade de informar a porta 3000.**
    
<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/rails2.png"/></th>
        </tr>
    </tbody>
</table>    
