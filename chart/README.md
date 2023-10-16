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
  |---rt
      |---data
          |---001.dat
          |---002.dat
          |---003.dat
          |---rt-serialized
  |---shredder
      |---20220XXX...-XXXX.sql
  |---RT_SiteConfig.pm
values.yaml
```

Ou seja, na raiz do repositório, só haverá o arquivo values.yaml e a pasta files. Dentro da pasta files ficarão todos as pastas e seus repectivos arquivos e subpastas.

## Arquivo `values.yaml`

Como o RT é configurado quase que totalmente por próprios arquivos de configuração e não variáveis de ambiente, por exemplo, o values.yaml possui apenas os seguintes campos a serem definidos:

### Persistência

```yaml
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

```yaml
service:
  rtPort: 9000
  type: ClusterIP
```

Em que:
- `rtPort`: porta utilizada pelo service
- `type`: tipo do service

OBS: deve-se atentar para qual porta está sendo definida no arquivo de configuração e colocar a mesma no service.

### Ingress

```yaml
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

```yaml
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

```yaml
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

# Pipeline
## Deploy
### RT
- Realiza o deploy do helm chart no namespace rt

```bash
helm upgrade rt . -i -n rt --create-namespace
```

## Config
### DB

- Configura o banco de dados postgres a partir das informações no arquivo `RT_SiteConfig.pm`

```
kubectl exec -n rt `kubectl get pods -o=name -n rt | grep rt-rt` -- bash /custom/postgres/init.sh
```

OBS: O script utilizado foi configurado no arquivo `values.yaml`:

```
scripts:
  init:
    perl /opt/rt5/sbin/rt-setup-database --action init --dba-password=123
```

Em que: 
- `init` indica quais os comandos dentro do script init.sh
- `dba-password`: senha para acessar o postgresql. OBS: mesma definida para o subchart do postgre e no arquivo de configuração `RT_SiteConfig.pm`

### Ldap users

- Importa os usuários do ldap configurado no arquivo `RT_SiteConfig.pm`

```
kubectl exec -n rt `kubectl get pods -o=name -n rt | grep rt-rt` -- bash /custom/postgres/ldap.sh
```

OBS: o script utilizado também foi configurado no arquivo `values.yaml`:

```yaml
scripts:
  ...
  ldap:
    rt-ldapimport --import --verbose 
```

- Além disso, no final, deleta o pod do rt (forma simples de resetar) para se conectar ao banco de dados agora já configurado

```bash
kubectl delete -n rt `kubectl get pods -o=name -n rt | grep rt-rt`
```

## Post Config
### Svc Account

Para que os cronjobs consigam acessar os outros pods para executar os scripts, é preciso que tenham um nível de permissão por meio de um service account e uma role associada. Portanto, esse job aplica os arquivos `svcaccount.yaml`, `role.yaml` e `role-binding.yaml` localizados na pasta `templates` para criar uma role e atribuí-la a um svc account por meio de uma role binding.

```bash
kubectl apply -f templates/role.yaml -n rt
kubectl apply -f templates/svcaccount.yaml -n rt
kubectl apply -f templates/role-binding.yaml -n rt
```

### Cronjobs
- Adiciona os cronjobs definidos a partir do arquivo `files/cron/crontab` e localizados na pasta `templates/conjobs`.

```bash
kubectl apply -f templates/cronjobs -n rt
```

### Cron

O RT precisa de uma rotina para obter os novos emails recebidos, então devem ser criados cronjobs do k8s configurados a partir do arquivo crontab, ou seja, esse arquivo não é necessário, apenas facilita na configuração desses cronjobs.

Porém, não é possível utilizar apenas os cronjobs, pois são temporários e não seria possível verificar certos recursos que precisam persistir. Por exemplo, os emails devem ser armazenados para que o rt veja se já foram lidos ou são novos, o que é necessário para o funcionamento correto do rt. 

Visto isso, há um deploy de um pod para realizar apenas essa comunicação rt e cronjobs, de modo que os cronjobs executam comandos nesse novo pod periodicamente.