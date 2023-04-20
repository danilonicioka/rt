# Helm Chart para Request Tracker (RT)

Baseado em [firefart/rt-docker](https://github.com/firefart/rt-docker), o qual utiliza a imagem [firefart/requesttracker](https://hub.docker.com/r/firefart/requesttracker). Portanto, os passos descritos abaixo para implementação desse serviço foram extraídos desse repositório e adaptados para o contexto do cluster.

Além disso, como o RT precisa de serviços em paralelo para seu funcionamento, alguns passos serão realizados para configurá-los e não o host do RT em si.

Um dos subcharts presentes é para o banco de dados PostgreSQL, baseado no helm chart do [cetic](https://artifacthub.io/packages/helm/cetic/postgresql). Os outros dois são para o cron, o qual usa a mesma imagem base do RT e para o nginx, o qual usa a imagem `firefart/requesttracker:nginx-nightly-20230321`

## Arquivos de configuração

Como explicado em [firefart/rt-docker](https://github.com/firefart/rt-docker#readme), são necessários alguns arquivos de configuração antes de iniciar o RT. Para o funcionamento correto do helm chart, deve-se colocar esses arquivos na seguinte estrutura no repositório gitlab:

```
files
  |---cron
      |---crontab
  |---getmail
      |---getmailrc
      |---certs
          |---pub.pem
  |---gpg
      |---crls.d
          |---DIR.txt
      |---pubring.kbx
      |---random_seed
      |---trustdb.gpg
  |---msmtp
      |---msmtp.conf
  |---nginx
      |---certs
          |---priv.pem
          |---pub.pem
      |---conf
          |---default.conf
          |---mailgate.conf
      |---startup-cripts
          |---script-x.sh
  |---rt-data
      |---001.dat
      |---002.dat
      |---003.dat
  |---shredder
      |---20220XXX...-XXXX.sql
  |---RT_SiteConfig.pm
values.yaml
```

Ou seja, na raiz do repositório, só haverá o arquivo values.yaml e a pasta files. Dentro da pasta files ficarão todos as pastas e seus repectivos arquivos e subpastas.

## Arquivo `values.yaml`

Como o RT é configurado quase que totalmente por próprios arquivos de configuração e não variáveis de ambiente, por exemplo, o values.yaml possui apenas os seguintes campos a serem definidos:

### Persistência

```
persistence:
  enabled: true
  storage: 500Mi
  storageClass: ""
  existingClaim: ""
```

Em que:
- `enabled`: define se a persistência está habilitada.
- `storage`: tamanho do armazenamento.
- `storageClass`: nome de um SC caso queira usar uma alternativa. Por padrão, será a `standard` do helm. 
- `existingClaim`: nome de um PVC caso queira usar um alternativo. Por padrão, será criado um novo.

### Service

```
service:
  rtPort: 9000
  type: ClusterIP
```

Em que:
- `rtPort`: porta utilizada pelo service
- `type`: tipo do service

OBS: deve-se atentar para qual porta está sendo definida no arquivo de configuração e colocar a mesma no service.

### Ingress

```
ingress:
  enabled: true
  className: ""
  host: rt-teste.com
  rt:
    path: /
    pathType: Prefix
    serviceName: rt-nginx
    servicePort: 80
```

Em que:
- `enabled`: define se o ingress será utilizado.
- `className`: nome de um ingress class alternativo. Por padrão, utilizará o ingress class definido como default no cluster.
- `host`: define hostname utilizado para acessar o RT
- `path`: define caminho para acessar RT
- `pathType: Prefix` indica que a rota deve corresponder ao prefixo indicado em path, ignora `/` após o prefixo, por exemplo. Ver mais na [documentação](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types)
- `serviceName`: nome do service que o ingress redirecionará as requisições
- `servicePort`: porta que o service está utilizando

### PostgreSQL

```
postgresql:
  persistence:
    enabled: true
    mountPath: /var/lib/postgresql/data
    subPath: data
    accessModes:  [ReadWriteOnce]
    storage: 10Gi
    existingClaim: ""
    storageClass: ""
  postgresql:
    username: postgres
    password: 123
    port: 5432
```

Em que:
- `mountPath`: diretório que será persistido 
- `subPath`: diretório em que o volume será montado
- `accessModes`: permissão de acesso ao conteúdo do volume
- `username`: usuário do postgresql
- `password`: senha do usuário
- `port`: porta que o postgresql utilizará

### Cron

```
cron:
  persistence:
    enabled: true
    data:
      storage: 100Mi
      existingClaim: ""
      storageClass: ""
    shredder:
      storage: 100Mi
      existingClaim: ""
      storageClass: ""
```

## Comandos para configuração

Para facilitar o processo de configuração após organizar todos os arquivos, pode-se utilizar alguns pequenos comandos/scrips listados a seguir:

OBS1: os seguintes passos são para os casos em que já exista uma instância do RT e deseja-se apenas migrá-la ou criar uma outra instância
OBS2: garantir que todos os pods estejam rodando
OBS3: devem ser executadas em ordem


1. Dados do RT

Caso ainda não tenha os dados de uma instância do RT, basta executar o seguinte comando:

```
rt-validator --check && rt-serializer --clone
```

Esse comando gerará uma pasta com os dados serializados do RT, logo, é preciso copiá-los para a nova instância.

Devido à limitação do tamanho do chart, não é possível fazer um mount diretamente, então deve-se realizar essa cópia de forma externa após o pod estar pronto. Isso pode ser feito por meio do comando abaixo:

```
kubectl cp files/rt-data `kubectl get pods --no-headers -o custom-columns=":metadata.name" | grep rt-rt`:/root/
```

2. Configuração do banco de dados

Antes de importar dados do RT, é preciso configurar o banco de dados de acordo com as informações anteriores para não ocorrer conflitos, ou seja, o nome do banco de dados, o usuário etc devem ser iguais aos utilizados na instância da qual se obteve esses dados.

Para isso, deve-se configurar corretamente o arquivo `RT_SiteConfig.pm` com essas informações. Após isso, pode-se utilizar o executável rt-setup-database para configurar o banco para ficar pronto para receber os dados.

Para facilitar esse processo, pode-se utilizar scripts com os comandos necessários e apenas executá-los, o que evitaria ter que acessar diretamente o container. Esses scripts podem ser configurados no arquivo `values.yaml`. Exemplo:

```
scripts:
  create:
    perl /opt/rt5/sbin/rt-setup-database --action create,schema,acl --dba-password=123
  import:
    perl /opt/rt5/sbin/rt-importer /root/rt-data
```

Em que: 
- `create` e `import` indicarão os comandos dentro dos scripts create.sh e import.sh, respectivamente
- `dba-password`: senha para acessar o postgresql. OBS: mesma definida no subchart do postgre e no arquivo de configuração `RT_SiteConfig.pm`

Desse modo, após definir a senha correta, pode-se executar os seguintes comandos:

```
kubectl exec -it `kubectl get pods -o=name | grep rt-rt` -- bash /custom/postgres/create.sh
```

```
kubectl exec -it `kubectl get pods -o=name | grep rt-rt` -- bash /custom/postgres/import.sh
```

3. LDAP

O RT pode importar usuários do LDAP para acessá-lo. Isso é feito por meio do executável rt-ldap-import.

Assim como no passo 2, esse comando pode ser descrito no arquivo `values.yaml` para ser colocado em um script. Exemplo:

```
scripts:
    ldap:
        rt-ldapimport --import --verbose 
```

OBS: deve-se verificar se a configuração para conectar ao ldap está corretamente configurada no arquivo RT_SiteConfig.pm

Após isso, basta executar o comando abaixo:

```
kubectl exec -it `kubectl get pods -o=name | grep rt-rt` -- bash /custom/postgres/ldap.sh
```

4. Reiniciar RT

Como o banco de dados foi reconfigurado, é preciso reiniciar o rt para que ele se conecte a esse banco. Para isso, pode-se deletar o pod com o comando abaixo, pois ele será levantado novamente de forma automática:

```
kubectl delete `kubectl get pods -o=name | grep rt-rt`
```