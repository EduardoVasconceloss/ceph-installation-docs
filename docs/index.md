## O que é o Ceph?

O Ceph é uma solução de armazenamento distribuído e descentralizado que utiliza um conjunto de servidores interconectados para criar um sistema de armazenamento altamente disponível e tolerante a falhas. Dito isso, vamos ver a instalação passo a passo de um cluster Ceph!

## Instale os pré-requisitos

A princípio, os pacotes necessários para a instalação do ceph serão somente dois. Entretanto, com o decorrer da instalação, vamos ter que instalar mais pacotes.

```bash
apt install -y curl gnupg
```

---

# Configurando o cluster Ceph e seus nodes

## 1. Gerando e configurando uma chave ssh nos Controllers nodes

As chaves SSH são necessárias para a conexão sem senha dos nodes entre si. Para gerar uma chave SSH e um arquivo de configuração, execute esses comandos no seu terminal

```bash
ssh-keygen
nano ~/.ssh/config
```

Dentro do arquivo de configuração criado, vamos determinar os hosts e hostnames dos nossos nodes

> **Nota:**
>
> ```bash
> 	# Defina cada node e SSH user
> Host controller1
> 	Hostname controller1
> 	User root
> Host controller2
> 	Hostname controller2
> 	User root
> Host controller3
> 	Hostname controller3
> 	User root
> ```

No seu terminal, faça a cópia da chave ssh do seu host aos outros nodes

> **Nota:**
>
> ```bash
> # Adicione a chave ssh do controller1 aos outros nodes e vice-versa
> ssh-copy-id controller1
> ssh-copy-id controller2
> ssh-copy-id controller3
> ```

## 2. Instale o ceph em cada node

Para instalar o ceph em cada node de maneira mais rápida, podemos utilizar a linguagem bash no terminal

```bash
for NODE in controller1 controller2 controller3
do
    ssh $NODE "apt update; apt -y install ceph ceph-common"
done
```

## 3. Configure o Monitor Daemon e o Maganer Daemon nos controllers nodes

Os monitores são responsáveis por manter e distribuir informações sobre o estado do cluster, já os managers são responsáveis por coletar, armazenar e forncecer informações sobre métricas e estatísticas do cluster.

Para configurar os monitores e managers, temos que criar um id específico e um arquivo de configuração para o cluster.

```bash
# comando para gerar um id para seu cluster
uuidgen

# crie um arquivo de configuração para o ceph
nano /etc/ceph/ceph.conf
```

Veja como deve ficar seu "ceph.conf"

> **Exemplo:**
>
> ```bash
> [global]
> fsid = {cluster uuid}
> mon initial members = {id1}, {id2}, {id2}
> mon host = {ip1}, {ip2}, {ip3}
> public network = {network range for your >public network}
> cluster network = {network range for your >cluster network}
> auth cluster required = cephx
> auth service required = cephx
> auth client required = cephx
> ```

## 4. Criando e importando as chaves para admin, monitor e bootstrap

Deve-se criar as "keyrings" para autenticação e permissão dos monitores, admin client e "bootstrap-osd". O comando "ceph-authtool" serve para manipular as "keyrings" de autenticação do ceph.

```bash
sudo ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'
sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
sudo chown ceph:ceph /etc/ceph/ceph.mon.keyring # Torna o usuário do ceph como o dono do arquivo "ceph.mon.keyring"
```

## 5. Crie um “monitor map” para os monitores se comunicarem e iniciando o monitor

É preciso ter um "mapa" para que os monitores possam se "enxergar", então iremos usar o comando "monmaptool" para criar esse "monitor map" e adicionar os monitores a esse mapa.

```bash
# Criando o mapa e adicionando os monitores
monmaptool --create --add {node1-id} {node1-ip} --fsid {cluster uuid} /etc/ceph/monmap
monmaptool --add {node2-id} {node2-ip} --fsid {cluster uuid} /etc/ceph/monmap
monmaptool --add {node3-id} {node3-ip} --fsid {cluster uuid} /etc/ceph/monmap

# Criando um diretório para armazenar os dados do monitor e incializando o monitor
sudo -u ceph mkdir /var/lib/ceph/mon/ceph-{node1-id}
sudo -u ceph ceph-mon --mkfs -i {node1-id} --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring
sudo systemctl start ceph-mon@{node1-id}
sudo systemctl enable ceph-mon@{node1-id}

# Ativando protocolos de comunicação e segurança para o cluster
sudo ceph mon enable-msgr2
ceph config set mon auth_allow_insecure_global_id_reclaim false
```

## 6. Criando um “manager” e habilitando o dashboard do ceph

Temos que criar credenciais de autenticação para o manager e garantir todas as permissões necessárias para ele

```bash
# Criando as credenciais e garantindo as permissões
sudo ceph auth get-or-create mgr.{node1-id} mon 'allow profile mgr' osd 'allow *' mds 'allow *'
sudo -u ceph mkdir /var/lib/ceph/mgr/ceph-{node1-id} # Copie a keyring que for 'printada' no seu terminal
sudo -u ceph nano /var/lib/ceph/mgr/ceph-{node1-id}/keyring # Cole a keyring que você copiou nesse arquivo

# Inicializando o manager
sudo systemctl start ceph-mgr@{node1-id}
sudo systemctl enable ceph-mgr@{node1-id}

# Instalando e habilitando o dashboard do ceph
apt -y install ceph-mgr-dashboard
sudo ceph mgr module enable dashboard
sudo ceph dashboard create-self-signed-cert
sudo nano passwd #coloque uma senha dentro desse arquivo
sudo ceph dashboard ac-user-create admin -i passwd administrator
```

## 7. Verificando o cluster via dashboard

No seu navegador, acesse o dashboard via ip do controller que você habilitou o dashboard

> **Nota:**
> https://SEUIP:8443/
> Usuário: admin
> Senha: "senha-que-você-adicionou-no-arquivo-passwd"

---

# Adicionando mais nodes “controllers” ao seu cluster

## 1. Transferindo os arquivos necessários para os outros nodes

Para adicionar mais nodes com monitor e manager, não precisamos fazer todo esse procedimento novamente para cada node. Basta enviarmos os arquivos de configuração, keyrings e o "monmap" para algum outro node.

```bash
sudo scp /etc/ceph/ceph.conf {user}@{server}:/etc/ceph/
sudo scp /etc/ceph/ceph.client.admin.keyring {user}@{server}:/etc/ceph/
sudo scp /var/lib/ceph/bootstrap-osd/ceph.keyring {user}@{server}:/var/lib/ceph/bootstrap-osd/
sudo scp /etc/ceph/monmap {user}@{server}:/etc/ceph/
sudo scp /etc/ceph/ceph.mon.keyring {user}@{server}:/etc/ceph/ceph.mon.keyring
```

## 2. No seu “controller2”, assim como no “controller1”, habilite o monitor e aplique esses seguintes passos:

É necessário fazer alguns ajustes nos arquivos que nós acabamos de transferir para que o monitor seja reconhecido no cluster

```bash
# Mudando o dono da keyring do monitor, criando um diretório para armazenar os dados e inicializando os dados do monitor
sudo chown ceph:ceph /etc/ceph/ceph.mon.keyring
sudo -u ceph mkdir /var/lib/ceph/mon/ceph-{node2-id}
sudo -u ceph ceph-mon --mkfs -i {node2-id} --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring

# Inicializando os serviços do monitor
sudo systemctl start ceph-mon@{node2-id}
sudo systemctl enable ceph-mon@{node2-id}

# Habilitando o protocolo de comunicação no monitor
sudo ceph mon enable-msgr2
```

---

# Criando OSDs para o cluster

## 1. Crie uma VM para servir como seu OSD e instale o ceph

É recomendado que você crie outra máquina virtual para "hostear" o seu OSD, já que é mais seguro e expande o seu cluster

```bash
apt update
apt -y install ceph ceph-common
```

## 2. Crie um disco específico para o OSD com o KVM

Para criar um OSD, é recomendado um disco com no mínimo 1 terabyte, porém, você pode criar um OSD com o tamanho que você quiser

```bash
# Crie o disco com esse comando
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/novodisco.qcow2 30G

# Adicione o disco a sua VM (Você pode fazer essa adição manualmente pelo KVM)
sudo virsh attach-disk minhaVM --source /var/lib/libvirt/images/novodisco.qcow2 --target vdb --persistent
```

## 3. No seu controller, copie os arquivos de configuração e mande para o node de OSD

Para adicionar o OSD ao cluster, precisamos enviar o arquivo de configuração, keyrings, bootstrap e o monmap para a VM do OSD.

```bash
sudo scp /etc/ceph/ceph.conf {user}@{server}:/etc/ceph/
sudo scp /etc/ceph/ceph.client.admin.keyring {user}@{server}:/etc/ceph/
sudo scp /var/lib/ceph/bootstrap-osd/ceph.keyring {user}@{server}:/var/lib/ceph/bootstrap-osd/
sudo scp /etc/ceph/monmap {user}@{server}:/etc/ceph/
sudo scp /etc/ceph/ceph.mon.keyring {user}@{server}:/etc/ceph/ceph.mon.keyring
```

## 4. No node de OSD, prepare o disco e inicie o OSD

Precisamos preparar o disco para ser usado como OSD usando os seguintes comandos:

```bash
sudo ceph-volume lvm prepare --data /dev/sdb
sudo ceph-volume lvm activate {osd-number} {osd-uuid}
sudo systemctl enable ceph-osd@{osd-number}
```

# Usando um Block Device no cluster

Um node de "block device" refere-se a um nó (ou servidor) no cluster que possui dispositivos de bloco anexados que são usados para armazenar dados. Esses dispositivos de bloco podem ser HDs ou SSDs que estão disponíveis no node e que são usados pelos demais componentes do Ceph, como OSDs, para armazenar os dados de maneira distribuída e replicada.

## 1. Criando um node "Ceph-Client"

Crie e configure uma VM nova para servir como um novo node “client” ao cluster. Depois, instale os pacotes necessários nessa VM.

```bash
apt -y install ceph-common xfsprogs
```

## 2. Enviando os arquivos de um node pra outro

No seu controller, copie os arquivos de configuração e mande para o node “client”

```bash
sudo scp /etc/ceph/ceph.conf {user}@{server}:/etc/ceph/
sudo scp /etc/ceph/ceph.client.admin.keyring {user}@{server}:/etc/ceph/
sudo scp /var/lib/ceph/bootstrap-osd/ceph.keyring {user}@{server}:/var/lib/ceph/bootstrap-osd/
sudo scp /etc/ceph/monmap {user}@{server}:/etc/ceph/
sudo scp /etc/ceph/ceph.mon.keyring {user}@{server}:/etc/ceph/ceph.mon.keyring
```

## 3. Criando uma pool

No seu client, mude o dono da key do ceph e crie uma pool rbd (rados block device). No Ceph, uma "pool" é uma partição lógica para armazenar dados. A pool 'rbd' é usada para armazenar imagens de dispositivos de bloco.

```bash
chown ceph. /etc/ceph/ceph.*
ceph osd pool create rbd 64
ceph mgr module enable pg_autoscaler
ceph osd pool set rbd pg_autoscale_mode on
rbd pool init rbd
```

## 4. Crie um block device

O RBD fornece uma interface de dispositivo de bloco em disco. Quando você cria uma imagem RBD, ela é dividida em múltiplos objetos no pool. Isso significa que o RBD é mapeado para um ou mais objetos em um cluster de armazenamento Ceph, e os dados do dispositivo de bloco são distribuídos por todo o cluster. Isso facilita o redimensionamento, a migração e a replicação dos dados.

```bash
rbd create --size 10G --pool rbd rbd01
rbd ls -l
rbd map rbd01
rbd showmapped
mkfs.xfs /dev/rbd0
mount /dev/rbd0 /mnt
df -hT
```

> **Atenção:**
> Para deletar o block device criado, siga esses passos:
> `bash

    rbd unmap /dev/rbd/rbd/rbd01
    rbd rm rbd01 -p rbd
    ceph osd pool delete rbd rbd --yes-i-really-really-mean-it
    `

# Usando um File System no cluster

O Ceph File System (CephFS) é um sistema de arquivos distribuído compatível com POSIX que usa um Ceph Storage Cluster para armazenar seus dados. O CephFS é um serviço robusto e totalmente equipado com recursos como snapshots, quotas e capacidades de espelhamento multi-cluster. Os arquivos do CephFS são distribuídos em objetos armazenados pelo Ceph, proporcionando alta escala e desempenho. Os sistemas Linux podem montar sistemas de arquivos CephFS nativamente, via um cliente baseado em FUSE, ou via um gateway NFSv4.

## 1. No node “client”, instale o “ceph-fuse”

O "ceph-fuse" permite montar o CephFS em um node client para que ele possa acessar e interagir com o sistema de arquivos distribuído do cluster Ceph.

```bash
ssh ceph-client "apt -y install ceph-fuse"
```

## 2. Configure o MDS (MetaData Server) em um node controller

O Metadata Server (MDS) é responsável por gerenciar os metadados do CephFS, como informações de diretórios, permissões e atributos.

```bash
mkdir -p /var/lib/ceph/mds/ceph-{node1-id}
ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-node01/keyring --gen-key -n mds.{node1-id}
chown -R ceph. /var/lib/ceph/mds/ceph-{node1-id}
ceph auth add mds.{node1-id} osd "allow rwx" mds "allow" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-{node1-id}/keyring
systemctl enable --now ceph-mds@{node1-id}
```

## 3. Crie 2 RADOS pools para Data e MetaData no node com MDS

Para o CephFS, você precisa criar duas pools, uma para armazenar os dados do sistema de arquivos (cephfs_data) e outra para armazenar os metadados (cephfs_metadata).

```bash
ceph osd pool create cephfs_data 16
ceph osd pool create cephfs_metadata 8
ceph fs new cephfs cephfs_metadata cephfs_data
ceph fs ls
ceph mds stat
ceph fs status cephfs
```

## 4. Monte uma partição CephFS no node Client

O comando ceph fs new cria um novo CephFS usando as duas pools criadas anteriormente para os metadados e os dados.

```bash
ceph-authtool -p /etc/ceph/ceph.client.admin.keyring > admin.key
chmod 600 admin.key
mount -t ceph node01.srv.world:6789:/ /mnt -o name=admin,secretfile=admin.key
df -hT
```

# Configurando o Ceph Object Gateway

o Ceph Object Gateway é um serviço que fornece interfaces compatíveis com a API S3 e API Swift, permitindo que aplicativos e clientes acessem objetos armazenados no cluster Ceph como se fossem serviços de armazenamento de objetos independentes.

## 1. Em um node especifico para Object Gateway, instale o radosgw

O radosgw fornece uma interface de armazenamento de objetos compatível com a API S3 (Amazon Simple Storage Service) e a API Swift (OpenStack Object Storage).

```bash
apt -y install radosgw
```

## 2. No node “controller1”, configure o ceph.conf

Vamos definir o host do node onde o Object Gateway está sendo instalado e selecionar a porta "8080" para civetweb, que é uma opção de frontend web para o Object Gateway.

```bash
# adicione ao final
# client.rgw.(NomeDoNode)
[client.rgw.NomeDoNode]
# IP do Node
host = 10.0.0.31
# selecione a porta
rgw frontends = "civetweb port=8080"
```

## 3. Transfira os arquivos de configuração

```bash
scp /etc/ceph/ceph.conf {user}@{server}:/etc/ceph/ceph.conf
scp /etc/ceph/ceph.client.admin.keyring {user}@{server}:/etc/ceph/
```

## 4. Configure o RADOSGW

Criando uma chave de autenticação específica para o client do Object Gateway e configurando os diretórios e permissões necessários.

```bash
ssh {user}@{server-do-RADOS-GW} \
"mkdir -p /var/lib/ceph/radosgw/ceph-rgw.$NomeDoNode; \
ceph auth get-or-create client.rgw.$NomeDoNode osd 'allow rwx' mon 'allow rw' -o /var/lib/ceph/radosgw/ceph-rgw.$NomeDoNode/keyring; \
chown ceph. /etc/ceph/ceph.*; \
chown -R ceph. /var/lib/ceph/radosgw; \
systemctl enable --now ceph-radosgw@rgw.$NomeDoNode"
```

## 5. Reinicie os nodes “controller1” e o node de radosgw

É recomendado reiniciar os nodes "controller1" e o node onde o Object Gateway foi configurado para garantir que todas as alterações de configuração entrem em vigor.

## 6. Verifique o status do RADOSGW

Acesse a interface através de uma chamada HTTP usando o "curl".

```bash
curl IPdoNode:8080
# Saída esperada:
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>
```

## 7. No node de object gateway, crie um usuário

Crie um usuário específico para o Object Gateway. Isso permitirá que você acesse o Object Gateway e armazene objetos usando a API S3.

```bash
# por exemplo, crie um usuário [tiex]
radosgw-admin user create --uid=tiex --display-name="Tiex"

# lista de usuários
radosgw-admin user list

# informação de usuário
radosgw-admin user info --uid=tiex
```

## 8. Use o script de teste em Python para verificar o acesso à interface S3

O script cria um bucket, lista os buckets existentes, remove o bucket criado e verifica o acesso bem-sucedido à interface S3.

```bash
# Instale o boto3
apt -y install python3-boto3

# crie o arquivo do script
nano s3_teste.py

# copie e cole no script
import sys
import boto3
from botocore.config import Config
# user's access-key and secret-key you added on [2] section
session = boto3.session.Session(
    aws_access_key_id = '{access_key_idDoUsuario}',
    aws_secret_access_key = '{secret_access_keyDoUsuario}'
)
# Object Gateway URL
s3client = session.client(
    's3',
    endpoint_url = 'http://10.0.0.31:8080',
    config = Config()
)
# create [my-new-bucket]
bucket = s3client.create_bucket(Bucket = 'my-new-bucket')
# list Buckets
print(s3client.list_buckets())
# remove [my-new-bucket]
s3client.delete_bucket(Bucket = 'my-new-bucket')

# execute o script
python3 s3_teste.py
```

# Instale o NFS-Ganesha

O NFS-Ganesha oferece a capacidade de compartilhar o CephFS com outros nodes na rede usando o protocolo NFS, permitindo que o CephFS seja acessado e usado como um sistema de arquivos compartilhado.

## 1. Instale e configure o NFS-Ganesha no node com CephFS

O NFS-Ganesha é um servidor NFS de usuário no espaço do usuário que suporta vários sistemas de arquivos back-end, incluindo o CephFS. Isso permitirá que você monte o CephFS usando o protocolo NFS.

```bash
apt -y install nfs-ganesha-ceph

mv /etc/ganesha/ganesha.conf /etc/ganesha/ganesha.conf.org

# Configurando o NFS-Ganesha para montar o CephFS usando o protocolo NFSv4.
nano /etc/ganesha/ganesha.conf

# copie e cole
NFS_CORE_PARAM {
    # desativa NLM
    Enable_NLM = false;
    # desativa RQUOTA (não suportado no CephFS)
    Enable_RQUOTA = false;
    # protocolo NFS
    Protocols = 4;
}
EXPORT_DEFAULTS {
    # acesso padrao
    Access_Type = RW;
}
EXPORT {
    # ID unico
    Export_Id = 101;
    # monta o caminho para CephFS
    Path = "/";
    FSAL {
        name = CEPH;
        # hostname ou IP do Node
        hostname="10.0.0.51";
    }
    # setando root Squash
    Squash="No_root_squash";
    # NFSv4 Pseudo caminho
    Pseudo="/vfs_ceph";
    # permitindo opções de segurança
    SecType = "sys";
}
LOG {
    # log level padrao
    Default_Log_Level = WARN;
}

systemctl restart nfs-ganesha
```

## 2. Verifique a montagem do NFS no node “Client” do ceph.

Vamos verificar se foi bem-sucedida a montagem do CephFS no node Client usando o protocolo NFSv4.

```bash
apt -y install nfs-common

mount -t nfs4 controller1:/vfs_ceph /mnt

df -hT
```

# Configurando ISCSI no ceph

O iSCSI (Internet Small Computer System Interface) é um protocolo que permite o uso da infraestrutura de rede TCP/IP para transporte de dados SCSI entre um servidor iSCSI (initiator) e um dispositivo de armazenamento (target). No contexto de Ceph, o iSCSI é usado para fornecer acesso a blocos de armazenamento do Ceph através de uma rede usando o protocolo iSCSI.

## 1. Crie uma pool para o iscsi

Pool no Ceph para armazenar as imagens do iSCSI.

```bash
sudo ceph osd pool create iscsi

sudo rbd pool init iscsi
```

## 2. Altere o arquivo de configuração do ceph

```bash
nano /etc/ceph/ceph.conf

# copie e cole
[global]
fsid = 9ea8b268-15fb-4f20-a6df-b40ded1eab11
mon initial members = controller1, controller2, controller3
mon host = 10.0.0.51, 10.0.0.52, 10.0.0.53
public network = 172.25.12.0/13
cluster network = 10.0.0.0/24
auth cluster required = cephx
auth service required = cephx
auth client required = cephx

[osd]
osd heartbeat grace = 20
osd heartbeat interval = 5
osd client watch timeout = 15

# client.rgw.(Node Name)
[client.rgw.ceph-radosgw]
# IP address of the Node
host = 10.0.0.31
# set listening port
rgw frontends = "civetweb port=8080"
```

## 3. Instale os seguintes pacotes em todos os nodes que terão o ISCSI

Esses pacotes são necessários para configurar o ISCSI.

```bash
sudo apt install -y targetcli-fb ceph-iscsi python3-rtslib-fb tcmu-runner
```

## 4. Configure o arquivo de configuração do ISCSI

```bash
nano /etc/ceph/iscsi-gateway.cfg

# copie e cole
[config]
cluster_name = ceph
gateway_keyring = ceph.client.admin.keyring
pool = iscsi

api_secure = false

#api_user = admin
#api_password = admin
#api_port = 5001
trusted_ip_list = 10.0.0.51,10.0.0.52,10.0.0.53,10.0.0.31,10.0.0.30,10.0.0.61,10.0.0.62,10.0.0.63,10.0.0.64,10.0.0.65,10.0.0.66,10.0.0.67,10.0.0.68,10.0.0.69
```

## 5. Reinicie os serviços do debian e habilite o serviço do gateway para iscsi

```bash
systemctl daemon-reload

systemctl enable rbd-target-gw

# lista a "blacklist"
ceph osd blacklist ls

# remove tudo da "blacklist"
ceph osd blacklist clear

systemctl enable rbd-target-api
systemctl start rbd-target-api
```

## 6. Adicionando os gateways ao cluster

```bash
sudo ceph dashboard set-iscsi-api-ssl-verification false

# lista os iscsi gateways existentes no cluster
sudo ceph dashboard iscsi-gateway-list

# crie um arquivo para cada iscsi gateway
micro gw1
micro gw2
micro gw3

# copie e cole nos arquivos criados
http://admin:admin@IPdoNode:5000

# adicione os gateways ao dashboard
sudo ceph dashboard iscsi-gateway-add -i gw1
sudo ceph dashboard iscsi-gateway-add -i gw2
sudo ceph dashboard iscsi-gateway-add -i gw3
```

---

# Licença

> **Desenvolvido pela equipe Tiex**

> **Autores:** Eduardo Vasconcelos

> **Data de criação:** 20/06/2023
