## PostgreSQL Master-Replica Setup with Pgpool-II

### Este documento descreve o processo de configuração de um ambiente com PostgreSQL Master-Replica utilizando Pgpool-II para balanceamento de carga e failover. O setup é realizado com Docker Compose, contendo um banco de dados principal (master), duas réplicas (replica-1 e replica-2) e o Pgpool-II.

#### Estrutura do Ambiente

postgres: Banco de dados principal (Master).

postgres-replica-1: Primeira réplica física do banco de dados.

postgres-replica-2: Segunda réplica física do banco de dados.

pgpool: Pgpool-II configurado para realizar o balanceamento de carga entre o Master e as réplicas.

Passo a Passo da Configuração

c 1. Configuração do Master (postgres)

postgres:
  image: postgres
  container_name: postgres-master
  environment:
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: password
    POSTGRES_HOST_AUTH_METHOD: "scram-sha-256\nhost replication all 0.0.0.0/0 md5"
  ports:
    - 5432:5432
  command: postgres
  shm_size: 1g
  volumes:
    - master_data:/var/lib/postgresql/data

Define o container postgres-master como banco de dados principal.

Configura o método de autenticação como scram-sha-256 e habilita a replicação.

Mapeia a porta 5432 do host para o container.

Usa o volume master_data para persistência de dados.

##### 2. Configuração das Réplicas

Replica 1 (postgres-replica-1)
postgres:
  image: postgres
  container_name: postgres-master
  environment:
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: password
    POSTGRES_HOST_AUTH_METHOD: "scram-sha-256\nhost replication all 0.0.0.0/0 md5"
  ports:
    - 5432:5432
  command: postgres
  shm_size: 1g
  volumes:
    - master_data:/var/lib/postgresql/data
    Define o container postgres-master como banco de dados principal.

Configura o método de autenticação como scram-sha-256 e habilita a replicação.

Mapeia a porta 5432 do host para o container.

Usa o volume master_data para persistência de dados.

##### 2. Configuração das Réplicas

Replica 1 (postgres-replica-1)
postgres-replica-1:
  image: postgres
  container_name: postgres-replica-1
  shm_size: 1g
  environment:
    PGUSER: replicator
    PGPASSWORD: replicator_password
    PGHOST: postgres
  command: |
    bash -c "
    PGUSER=postgres PGPASSWORD=password psql -c \"CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'replicator_password';\";  
    PGUSER=postgres PGPASSWORD=password psql -c \"SELECT pg_create_physical_replication_slot('replication_slot');\";  
    rm -rf /var/lib/postgresql/data/* && pg_basebackup --pgdata=/var/lib/postgresql/data -R --slot=replication_slot;
    chown -R postgres:postgres /var/lib/postgresql/data && chmod 0700 /var/lib/postgresql/data;
    su postgres -c postgres;
    "
  depends_on:
    - postgres
  restart: always
  ports:
    - 5433:5432
  volumes:
    - replica1_data:/var/lib/postgresql/data

    Cria o usuário replicator com permissão de replicação.

Cria um slot de replicação físico chamado replication_slot.

Realiza um backup base usando pg_basebackup e configura a réplica com o slot criado.

##### Replica 2 (postgres-replica-2)
postgres-replica-2:
  image: postgres
  container_name: postgres-replica-2
  shm_size: 1g
  environment:
    PGUSER: replicator2
    PGPASSWORD: replicator_password
    PGHOST: postgres
  command: |
    bash -c "
    PGUSER=postgres PGPASSWORD=password psql -c \"CREATE USER replicator2 WITH REPLICATION ENCRYPTED PASSWORD 'replicator_password';\";  
    PGUSER=postgres PGPASSWORD=password psql -c \"SELECT pg_create_physical_replication_slot('replication_slot2');\";  
    rm -rf /var/lib/postgresql/data/* && pg_basebackup --pgdata=/var/lib/postgresql/data -R --slot=replication_slot2;
    chown -R postgres:postgres /var/lib/postgresql/data && chmod 0700 /var/lib/postgresql/data;
    su postgres -c postgres;
    "
  depends_on:
    - postgres
  restart: always
  ports:
    - 5434:5432
  volumes:
    - replica2_data:/var/lib/postgresql/data

    Similar à postgres-replica-1, mas cria o usuário replicator2 e o slot replication_slot2.

   ##### 3. Configuração do Pgpool-II (pgpool)
   pgpool:
  image: bitnami/pgpool:4
  container_name: pgpool
  environment:
    PGPOOL_POSTGRES_USERNAME: postgres
    PGPOOL_POSTGRES_PASSWORD: password
    PGPOOL_BACKEND_NODES: >-
      0:postgres:5432,
      1:postgres-replica-1:5433,
      2:postgres-replica-2:5434
    PGPOOL_ENABLE_LOAD_BALANCING: "yes"
    PGPOOL_SR_CHECK_USER: postgres
    PGPOOL_SR_CHECK_PASSWORD: password
    PGPOOL_ADMIN_USERNAME: postgres
    PGPOOL_ADMIN_PASSWORD: ppassword
  ports:
    - "9999:5432"
  depends_on:
    - postgres
    - postgres-replica-1
    - postgres-replica-2

Configura o Pgpool-II com os nós de backend: Master e Réplicas.

Habilita o balanceamento de carga entre as réplicas.

Configura o usuário de checagem de replicação.

Define a porta 9999 para acesso ao Pgpool-II.

##### 4. Volumes Persistentes
master_data: Volume para armazenar os dados do banco principal.

replica1_data: Volume para armazenar os dados da primeira réplica.

replica2_data: Volume para armazenar os dados da segunda réplica.

#### Como Executar

Salve o arquivo acima como docker-compose.yml.

Execute o comando: docker-compose up -d

Aguarde a inicialização completa dos containers.

Acesse o Pgpool-II na porta 9999 do host local.

#### Verificação do Ambiente

Conecte-se ao Pgpool-II usando um cliente PostgreSQL: psql -h localhost -p 9999 -U postgres -W

Verifique o status dos nós de backend: SHOW pool_nodes;