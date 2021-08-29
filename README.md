# GitHub Actions e RoR no OCI

## Introdução

O procedimento apresentado nesta documentação descreve os passos para o instanciar uma (ou mmais) VM dentro do Oracle Cloud e executar um workload em Ruby On Rail. O provisionamento será realizado desde o pipeline do GitHub actions.

## Pré-Requisitos

### Conta no Oracle Cloud

Para criar Config File, ingresse ao Oracle Cloud, vá a Identidadee/Segurnaça >  Usuário, selecione o usuário que terá acesso, crie uma API KEY> Solicite para que o public e private PEM sejam gerados junto ao processo.

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/api_key.png"/></th>
        </tr>
    </tbody>
</table>


### Conta no GitHub

Foi criado o script oci.yml dentro do GitHub Action que necessita de três segredos: CONFIG, OCI_KEY_FILE, ID_RSA_PRIV e ID_RSA

```
echo "${{secrets.CONFIG}}" >> ~/.oci/config
echo "${{secrets.OCI_KEY_FILE}}" >> ~/.oci/key.pem
echo "${{secrets.ID_RSA}}" >> ~/.oci/id_rsa.pub
```
Para gerar o secret no git, em seu projeto vá a Setting, depois Secrets. Clique no botão "New Repository Secret" para criar as 3 secrets.

<table>
    <tbody>
        <tr>
        <th><img align="left" width="600" src="https://objectstorage.us-ashburn-1.oraclecloud.com/n/idsvh8rxij5e/b/imagens_git/o/action_secret_git.png"/></th>
        </tr>
    </tbody>
</table>



Fique atento ao path de key_file que deverá ser informado no valor do segredo do CONFIG. O arquivo gerado na criaçãod a API KEY anterior terá o seguinte formato:

```
[DEFAULT]
user=ocid1.user.oc1..aaaaaaaa3wqzj7nxxfkrnz7y4ewt.....
fingerprint=51:4f:c2:79:b3:78:7a:48:89:2f:01:97:11:01:f4:e0
tenancy=ocid1.tenancy.oc1..aaaaaaaaqqzek25x6.....
region=us-ashburn-1
key_file=~/.oci/key.pem
```




