## Criando Cluster para o PostgreSQL

Obs: Foi utilizado a versão do postgresql 12.22
Obs: Para cada comando será colocado o sinal de $ na frente do comando, terá caso que sao varios comando seguidos e tem que ser executado um por vez.

Faremos 2 servidores um Master e um Slave, onde o master será o servidor principal e o slave sera a cópia , a instalação será feita compilando o source e o procedimento de instalação terá que ser feito  em ambos servidores


### Instalação do PostgreSQL

- Verificar o locale da maquina com o comando
```bash
$ locale
```
<br>
Deverá ter esse retorno

```sh
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR:pt:en
LC_CTYPE="pt_BR.UTF-8"
LC_NUMERIC="pt_BR.UTF-8"
LC_TIME="pt_BR.UTF-8"
LC_COLLATE="pt_BR.UTF-8"
LC_MONETARY="pt_BR.UTF-8"
LC_MESSAGES="pt_BR.UTF-8"
LC_PAPER="pt_BR.UTF-8"
LC_NAME="pt_BR.UTF-8"
LC_ADDRESS="pt_BR.UTF-8"
LC_TELEPHONE="pt_BR.UTF-8"
LC_MEASUREMENT="pt_BR.UTF-8"
LC_IDENTIFICATION="pt_BR.UTF-8"
LC_ALL=
```

Caso o LANG não for pt_BR.UTF-8 deverá executar o comando
```bash
$ 	dpkg-reconfigure locales
```
e escolher a opção pt_BR.UTF-8, após isso reiniciar a maquina.

- Criando os PATH do Linux
```bash
$ PATH=$PATH:/banco/postgres/pgsql/bin
$ cat >> /root/.bashrc << "EOF"
export LS_OPTIONS='--color=auto'
eval "`dircolors`"
alias ls='ls $LS_OPTIONS'
alias ll='ls $LS_OPTIONS -l'
alias l='ls $LS_OPTIONS -ltrahF'
alias psa='ps -auxf'
PATH=$PATH:/banco/postgres/pgsql/bin
EOF
```

- Instalar dependencias pacotes

```bash
$ apt update
$ apt install -y gcc g++ make automake autoconf libreadline-dev zlib1g-dev libstdc++5 libsystemd-dev bison flex libcurl4-openssl-dev libxml2-dev sqlite3 zip libncurses5-dev binutils-dev build-essential libasound2 dirmngr p7zip-full debian-archive-keyring gnupg
```

- Criar os diretórios e o usuario do postgresql e da permissão.
```bash
$ mkdir -p /banco/{postgres/pgsql/data,trash}

$ mkdir -p /home/postgres
$ useradd -d /home/postgres postgres
$ chown -v postgres:postgres /home/postgres
$ chown -v postgres:postgres /banco/postgres/pgsql/data
```

- Fazer o download dos fontes do PostgreSQL
```bash
$ cd /usr/src/
$ wget --no-check-certificate https://ftp.postgresql.org/pub/source/v12.22/postgresql-12.22.tar.gz
```

- Descompactar os fontes
```bash
$ tar xf postgresql-12.22.tar.gz
$ cd postgresql-12.22/
```

- Compilando o PostgreSQL
```bash
$ ./configure --with-systemd --prefix /banco/postgres/pgsql
$ make all && make install
$ cd contrib
$ make && make install
```

- Criando a base de dados
```bash
$ su -l postgres -c "/banco/postgres/pgsql/bin/initdb -D /banco/postgres/pgsql/data"
```

- Criando a inicialização automatica
```bash
$ cat > /lib/systemd/system/postgresql.service << "EOF"
[Unit]
Description=PostgreSQL database server
Documentation=man:postgres(1)

[Service]
Type=notify
User=postgres
ExecStart=/banco/postgres/pgsql/bin/postgres -D /banco/postgres/pgsql/data
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=0

[Install]
WantedBy=multi-user.target
EOF

$ systemctl enable postgresql.service
$ systemctl start postgresql.service
```

- Alterando a senha do usuário postgres
```bash
$ psql -p 5432 -U postgres template1 -c "ALTER USER postgres WITH PASSWORD '123456';"
```
<br>
<br>

## Configurando o servidor Master

- Criando usuário de replicação
```bash
$ psql -c "CREATE USER replication REPLICATION LOGIN CONNECTION LIMIT 1 ENCRYPTED PASSWORD '654321';"
```

- Configurando o arquivo pg_hba.conf
```bash
$ sed -ie "/# IPv4 local.*/a host	all		all		0.0.0.0/0	md5" /banco/postgres/pgsql/data/pg_hba.conf

$ sed -ie "/# replication privilege.*/a host    replication     replication     MasterIP/24         md5" /banco/postgres/pgsql/data/pg_hba.conf

$ sed -ie "/# replication privilege.*/a host    replication     replication     SlaveIP/24         md5" /banco/postgres/pgsql/data/pg_hba.conf
```
Obs: Trocar o MasterIP pelo ip da maquina Master e o SlaveIP pelo ip da maquina Slave.
<br>

- Configurando o arquivo postgresql.conf
```bash
$ sed -i -e "s/#listen_add.*/listen_addresses = '*'/" -e \
		 "s/#port.*/port = 5432/" -e \
		 "s/#enable_seqscan.*/enable_seqscan = off/" -e \
		 "s/lc_messages.*/#lc_messages = 'pt_BR.UTF-8'/" -e \
		 "s/lc_monetary.*/#lc_monetary = 'pt_BR.UTF-8'/" -e \
		 "s/lc_numeric.*/#lc_numeric = 'pt_BR.UTF-8'/" -e \
		 "s/lc_time.*/#lc_time = pt_BR.UTF-8/" -e \
		 "s/#wal_level.*/wal_level = replica/" -e \
		 "s/#wal_keep_segments.*/wal_keep_segments = 64/" -e \
		 "s/#max_wal_senders.*/max_wal_senders = 10/" -e \
		 "s/#default_with_oids.*/default_with_oids = off/" /banco/postgres/pgsql/data/postgresql.conf
```

- Reiniciando o serviço do PostgreSQL
```bash
$ systemctl restart postgresql
```
<br>
<br>

## Configurando o servidor Slave

 - Parando o serviço do PostgreSQL
```bash
$ systemctl stop postgresql
```

- Apagando a base de dados
```bash
$ rm -rf /banco/postgres/pgsql/data/*
```

- Copiando a base da dados do master
```bash
$ pg_basebackup -h `MasterIP` -D /banco/postgres/pgsql/data/ -P -U
replication --wal-method=fetch
```
Obs: Trocar o MasterIP pelo ip da maquina Master.
<br>
- Editar o arquivo postgresql.conf
```bash
$ sed -i -e "s/#listen_add.*/listen_addresses = '*'/" -e \
		 "s/#port.*/port = 5432/" -e \
		 "s/#enable_seqscan.*/enable_seqscan = off/" -e \
		 "s/lc_messages.*/#lc_messages = 'pt_BR.UTF-8'/" -e \
		 "s/lc_monetary.*/#lc_monetary = 'pt_BR.UTF-8'/" -e \
		 "s/lc_numeric.*/#lc_numeric = 'pt_BR.UTF-8'/" -e \
		 "s/lc_time.*/#lc_time = pt_BR.UTF-8/" -e \
		 "s/#wal_level.*/wal_level = replica/" -e \
		 "s/#wal_keep_segments.*/wal_keep_segments = 64/" -e \
		 "s/#max_wal_senders.*/max_wal_senders = 10/" -e \
		 "s/#hot_standby.*/hot_standby = on/" -e \
		 "s/#default_with_oids.*/default_with_oids = off/" /banco/postgres/pgsql/data/postgresql.conf
```

- Criando o arquivo recovery.conf
```bash
$ cat > /banco/postgres/pgsql/data/recovery.conf << "EOF"
standby_mode = 'on'
primary_conninfo = 'host=MasterIP port=5432 user=replication password=654321'
trigger_file = '/tmp/MasterNow'
EOF
```
Obs: Trocar o MasterIP pelo ip da maquina Master.
<br>
- Startando o serviço do PostgreSQL
```bash
$ systemctl start postgresql
```
