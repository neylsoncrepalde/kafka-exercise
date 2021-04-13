# Kafka Exercise

### Prof. Neylson Crepalde

#### Março de 2021

Exercício para praticar uma pipeline de Streaming de Dados com Kafka. Vamos implementar a seguinte arquitetura:

Integração do Kafka com uma database (postgresql) usando *kafka connect*, processamento de dados em streaming com **ksqlDB* e entrega em data lake com *kafka connect*.

**Colar desenho da arquitetura aqui!**

---

# Passo a passo para replicação (execução local)

## 1 - Configurar a Confluent Platform para executar localmente

...

## 2 - Instalar o cliente da confluent-hub

...

## 3 - Configurar variáveis de ambiente

É necessário definir a variável que contém o *path* para o home do Kafka.

```bash
export CONFLUENT_HOME=/home/<seu-usuario>/confluent-6.1.1
```

## 4 - Iniciar o docker container com o PostgreSQL

```bash
docker run --name test-postgres -e POSTGRES_PASSWORD=<sua-senha> -p 5432:5432 --rm -d postgres:13.2
```

## 5 - Executar o gerador de dados *fake*

```bash
python make_fake_data.py
```

O módulo `make_fake_data.py` possui 4 argumentos que podem ser utilizados na linha de comando. Acrescente `--interval` para definir quantos segundos o simulador vai aguardar entre as simulações, `-n` para definir quantos casos serão simulados por vez, `--connection-string` ou `-cs` para definir uma string de conexão customizada (para reaproveitamento do módulo em outra database) e `--silent` caso não desejemos exibir os dados simulados na tela.

Abaixo, a documentação do comando

    usage: make_fake_data.py [-h] [--interval INTERVAL] [-n N]
                         [--connection-string CONNECTION_STRING]
                         [--silent [SILENT]]

    Generate fake data...

    optional arguments:
    -h, --help            show this help message and exit
    --interval INTERVAL   interval of generating fake data in seconds
    -n N                  sample size
    --connection-string CONNECTION_STRING, -cs CONNECTION_STRING
                            Connection string to the database
    --silent [SILENT]     print fake data

Será necessário executar o simulador apenas uma vez para criar a tabela na database.

## 6 - Criar um tópico no Kafka

Vamos criar um tópico no kafka que irá armazenar os dados movidos da fonte.

```bash
kafka-topics --create \
    --bootstrap-server localhost:9092 \
    --partitions 1 \
    --replication-factor 1 \
    --topic psg-customers
```

O sufixo do nome do tópico deve possuir o mesmo nome da tabela cadastrado no arquivo `make_fake_data.py` caso seja necessário customizar.

## 7 - Registrar os parâmetros de configuração do connector no kafka

Para isso, vamos precisar de um arquivo `json` contendo as configurações do conector que vamos registrar. O arquivo `connect_postgres.config` possui um exemplo de implementação. O conteúdo do arquivo está transcrito abaixo:

```json
{
    "name": "postg-connector",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "tasks.max": 1,    
        "connection.url": "jdbc:postgresql://127.0.0.1:5432/postgres",
        "connection.user": "postgres",
        "connection.password": "SUA-SENHA",
        "mode": "timestamp",
        "timestamp.column.name": "dt_update",
        "table.whitelist": "public.customers",
        "topic.prefix": "psg-",
        "validate.non.null": "false",
        "poll.interval.ms": 500
    }
}
```

Com o arquivo, fazemos uma chamada à API do Kafka para registrar os parâmetros:

```bash
curl -X POST -H "Content-Type: application/json" \
    --data @connect_postgres.config http://localhost:8083/connectors
```

Este comando cria um conector que irá puxar todo o conteúdo da tabela mais todos os novos dados que forem inseridos. **Atenção**: O Kafka connect não puxa, por default, alterações feitas em registros já existentes. Puxa apenas novos registros. Para verificar se nossa configuração foi criada corretamente e o conector está ok, vamos exibir os logs. Utilizando a CLI da Confluent, faça

```bash
confluent local services connect log
```

e verifique se não há nenhuma mensagem de erro. 

## 8 - Iniciar um stream no ksqlDB

Para iniciar o ksqlDB apontando os logs da aplicação para uma pasta melhor localizada, fazemos

```bash
LOG_DIR=$CONFLUENT_HOME/ksql_logs ksql
```

Antes de começar, vamos conferir se nosso tópico foi criado corretamente. Na *CLI* do ksqlDB, faça

```
ksql> show topics;
```

    Kafka Topic                 | Partitions | Partition Replicas 
    ---------------------------------------------------------------
    default_ksql_processing_log | 1          | 1                  
    psg-customers               | 1          | 1                  
    ---------------------------------------------------------------

para mostrar os tópicos criados. Para verificar se nosso conector está rodando corretamente, podemos fazer

```
ksql> show connectors;
```

    Connector Name  | Type   | Class                                         | Status                      
    -------------------------------------------------------------------------------------------------------
    psg-connector   | SOURCE | io.confluent.connect.jdbc.JdbcSourceConnector | RUNNING (1/1 tasks RUNNING) 
    -------------------------------------------------------------------------------------------------------

O Status deve estar como RUNNING.

OK! O Kafka agora está puxando dados da tabela e registrando no tópico `psg-customers`. Podemos conferir o fluxo de dados no tópico com

```
ksql> print psg-customers;
```

Isso vai exibir as mensagens como ficam registradas no tópico. Para consumir o dado de uma maneira mais interessante, podemos criar um STREAM:

```
ksql> create stream custstream WITH (kafka_topic='psg-customers', value_format='AVRO');
```

Após a mensagem de confirmação, podemos verificar o stream assim:

```
ksql> show streams;
```

    Stream Name         | Kafka Topic                 | Key Format | Value Format | Windowed 
    ------------------------------------------------------------------------------------------
    CUSTSTREAM          | psg-customers               | KAFKA      | AVRO         | false    
    KSQL_PROCESSING_LOG | default_ksql_processing_log | KAFKA      | JSON         | false    
    ------------------------------------------------------------------------------------------

Para fazer uma consulta rápida ao stream (apenas exibí-lo na tela), podemos fazer

```
ksql> select * from custstream emit changes;
```

O output dessa consulta não é dos melhores. Além de conter um número grande de colunas, dificultando a visualização, todas as colunas de data estão no formato BIGINT o que não é nada intuitivo para interpretação. Podemos fazer uma consulta mais enxuta e já corrigindo essas datas com o seguinte código:

```
ksql> select nome, telefone, email, 
>TIMESTAMPTOSTRING(nascimento, 'yyyy-MM-dd', 'UTC') as dt_nascimento,
>TIMESTAMPTOSTRING(dt_update, 'yyyy-MM-dd HH:mm:ss.SSS', 'UTC') as dt_updt_conv
>from custstream emit changes;
```

A consulta retorna a seguinte tabela:

    +-------------------------+-------------------------+--------------------------+-------------------------+-------------------------+
    |NOME                     |TELEFONE                 |EMAIL                     |DT_NASCIMENTO            |DT_UPDT_CONV             |
    +-------------------------+-------------------------+--------------------------+-------------------------+-------------------------+
    |Scott Johnson            |+1-475-559-2163x6531     |michelle01@example.org    |1970-01-01               |2021-04-12 23:26:03.655  |
    |Amy Shannon              |902-547-6469             |michaelrogers@example.com |1969-12-31               |2021-04-12 23:26:04.211  |
    |Julie Kane               |+1-817-150-3155          |austin36@example.net      |1970-01-01               |2021-04-12 23:26:04.758  |
    |Sheri Fuller             |9634802570               |savannahduncan@example.net|1969-12-31               |2021-04-12 23:26:05.285  |

Bem mais interessante!

