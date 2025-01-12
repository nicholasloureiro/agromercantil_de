# 1- Cenário de Banco de Dados (DBA) #

Sua empresa possui um banco de dados PostgreSQL com uma tabela de logs que cresce em 1 milhão de linhas por dia. Atualmente, as consultas realizadas nessa tabela estão extremamente lentas.

## Tarefas: #
Proponha uma solução que otimize a performance para consultas de logs recentes (últimos 7 dias) sem degradar a consulta de logs antigos.

Explique como você implementaria:

Particionamento de tabelas

Estratégia de indexação

Backup e manutenção regular da tabela.

## Resposta #

Seguindo as orientações da pergunta, eu até poderia particionar a tabela de logs por intervalo de data, criar índices baseados no ID do log ou na sua data de criação e realizar backups incrementais diários e completos semanais para garantir a integridade dos dados. Mas, a longo prazo, isto seria uma má solução para este problema, e já está sendo, pois foi dito que as consultas estão extremamente lentas, tendo em vista que a tabela só iria crescer, as soluções de indexação e particionamento seriam apenas momentâneas e nada escaláveis, além do custo de manter uma massa de dados deste tamanho no banco. Se for no RDS, por exemplo:

### Estimativa de custo

Vou supor que cada log tenha um tamanho médio de 1 KB (1000 bytes).

Volume de Dados Mensal:

1 milhão de linhas/dia * 1 KB/linha * 30 dias = **30 GB/mês**

### Custo do RDS:

O custo do RDS depende da instância escolhida. Vou considerar uma instância db.t2.medium, que custa aproximadamente **R$ 0,15** por hora.

### Custo de Armazenamento:

O custo de armazenamento no RDS é de aproximadamente **R$ 0,02 por GB/mês.**

### Custo de Backup:

O custo de backup incremental é de aproximadamente **R$ 0,01 por GB/mês.**

### Total Estimado

Custo da Instância RDS: 24 horas/dia * 30 dias * R$ 0,15/hora = **R$ 108/mês**

Custo de Armazenamento: 30 GB * R$ 0,02/GB = **R$ 0,60/mês**

Custo de Backup: 30 GB * R$ 0,01/GB = **R$ 0,30/mês**

Total Estimado: R$ 108 + R$ 0,60 + R$ 0,30 = **R$ 108,90/mês**


--------------------------------------------------------------------------------


Minha primeira ação seria perguntar se há **necessidade** de manter esses logs no banco de dados e qual a frequência de acesso a esses dados, de acordo com o período. Por exemplo, se o acesso é frequente somente dentro do período de 7 dias, é interessante tirar os dados mais antigos do banco e movê-los para algum armazenamento preparado para acomodar uma massa de dados deste tamanho, melhorando a performance da tabela, **que fica menor**, e os custos, que ficam **menores** também.

Caso fosse confirmado que os logs depois de 7 dias são raramente acessados, eu utilizaria o recurso de exportação de dados do PostgreSQL para exportar logs antigos periodicamente para um bucket S3 e faria a configuração das políticas de ciclo de vida para otimizar os custos. As políticas de ciclo de vida são os tipos de armazenamento dentro do S3; dados que são acessados com menos frequência são naturalmente mais baratos de armazenar. Porém, há algumas opções que possuem delay para apresentar os dados, por isso é importante confirmar se a frequência de acesso muda de acordo com a "idade" do log e qual a urgência de obter dados mais antigos, tendo em vista que há opções com delays de até 12 horas, e que quanto maior a espera, menor é o custo da opção.

Eu pensei em duas políticas de ciclo de vida interessantes para este caso:


1. S3 Glacier Instant Retrieval para manter os logs por 30 dias, com delay de acesso de milissegundos.

2. S3 Glacier Flexible Retrieval para manter os logs até 90 dias, com delay de acesso entre 1 minuto e 12 horas. Após 90 dias, os logs seriam excluídos.

### Estimativa de Custo ###

Vou supor que cada log tenha um tamanho médio de 1 KB.
### Volume de Dados Mensal: ###

1 milhão de linhas/dia * 1 KB/linha * 30 dias = **30 GB/mês**

### Custo do S3 Glacier Instant Retrieval:

O custo é de aproximadamente **R$ 0,004** por GB/mês.

Para 30 GB: 30 GB * R$ 0,004/GB = **R$ 0,12/mês**

Custo do S3 Glacier Flexible Retrieval:

O custo é de aproximadamente **R$ 0,0036** por GB/mês.

Para 30 GB: 30 GB * R$ 0,0036/GB = **R$ 0,108/mês**

### Total Estimado
Primeiros 30 dias (S3 Glacier Instant Retrieval): R$ 0,12/mês

De 31 a 90 dias (S3 Glacier Flexible Retrieval): R$ 0,108/mês


## REDUÇÃO DE CUSTOS DE APROXIMADAMENTE 98%

Ref: https://aws.amazon.com/pt/s3/pricing/?gclid=CjwKCAiA7Y28BhAnEiwAAdOJUHkHk7-4SguNMVbmbXqOCnowlx4VJ56qu33bwgQvq43Ra_tL1m5-ABoChmsQAvD_BwE&trk=3daf7be6-997a-4e7c-af79-be4074c210e4&sc_channel=ps&ef_id=CjwKCAiA7Y28BhAnEiwAAdOJUHkHk7-4SguNMVbmbXqOCnowlx4VJ56qu33bwgQvq43Ra_tL1m5-ABoChmsQAvD_BwE:G:s&s_kwcid=AL!4422!3!536456042828!p!!g!!amazon%20s3%20data%20storage%20pricing!12024810840!115492236145

Ref: https://aws.amazon.com/pt/rds/aurora/pricing/

----------------------------------------------------------------------------------------------------------------------------------------

# 2- Pipeline de Dados

Você está criando um pipeline que deve:
1. Ler dados brutos de sensores IoT armazenados no S3.
2. Processar os dados em lotes (batch processing).
3. Carregar os resultados no Redshift para análises futuras.

## Tarefas:
- Desenhe um diagrama arquitetural que represente o pipeline.
- Liste os principais desafios técnicos que você enfrentaria e como resolveria cada um.
- Escreva um script em Python que implemente a etapa de leitura do S3 e aplicação de uma transformação simples (ex.: cálculo de média de um campo).

## Resposta

![image](https://github.com/user-attachments/assets/fdbf6dbd-d227-4eb7-a61f-3d56e0dbdb5e)

No diagrama acima, todos os dados coletados dos sensores IoT são centralizados no Amazon S3. Utilizando a funcionalidade de evolução de schema do AWS Glue, o Amazon Redshift Spectrum pode lidar automaticamente com mudanças no schema dos dados, como a adição ou remoção de colunas (por mais que haja alterações de estrutura, o pipeline não quebraria). Isso é alcançado através de um crawler do AWS Glue, que lê e adapta as mudanças de schema com base nas estruturas dos arquivos no S3. O crawler cria um schema híbrido, compatível com datasets antigos e novos. Assim, todos os arquivos de dados ingeridos podem ser lidos de um local especificado no S3 por meio de uma única tabela do Amazon Redshift Spectrum, referenciando o catálogo de metadados do AWS Glue.

Para executar a tarefa, eu carregaria os dados iniciais no S3 e executaria um AWS Glue Crawler para catalogá-los. Depois, criaria um schema externo no Redshift e usaria o Redshift Spectrum para consultar a tabela e ler os dados iniciais.

Adicionaria novos elementos ao template do KDG e enviaria os dados ao Firehose, validando a ingestão no S3 e atualizando as definições de tabela com o Glue Crawler. Consultaria a tabela no Redshift Spectrum para ler os dados combinados de dois schemas. Removeria uma coluna do template, enviaria ao Firehose, validaria a ingestão no S3 e atualizaria as definições com o Glue Crawler.

### Principais Desafios Técnicos e Soluções

1. **Coleta de Dados em Tempo Real**:
   - **Desafio**: Lidar com a grande quantidade de dados gerados pelos sensores IoT.
   - **Solução**: Utilizar o Amazon S3 para armazenar os dados brutos, proporcionando um armazenamento eficiente e escalável. (Também aplicando as políticas de ciclo de vida cabíveis para o tipo de uso)

2. **Transformação de Dados**:
   - **Desafio**: Processar dados em tempo real e realizar transformações complexas.
   - **Solução**: Utilizar o AWS Glue para ETL, **sem a necessidade de intervenção manual**. e Amazon Data Firehose para a ingestão dos dados transformados.

3. **Armazenamento e Consulta**:
   - **Desafio**: Armazenar e consultar grandes volumes de dados para análise futura.
   - **Solução**: Carregar os dados transformados no Amazon Redshift, que é otimizado para consultas analíticas.

4. **Gerenciamento de Acessos**:
   - **Desafio**: Garantir a segurança e gerenciamento adequado dos dados.
   - **Solução**: Utilizar o AWS Identity and Access Management (IAM) para controlar acessos e permissões.
## Script Python
```python
import boto3
import pandas as pd
from io import StringIO


s3_client = boto3.client('s3')
bucket_name = 'bucketagromercantil'
file_key = 'agromercantil/sensores/dados_csv/arquivo.csv'


def read_s3_data(bucket, key):
    response = s3_client.get_object(Bucket=bucket, Key=key)
    data = response['Body'].read().decode('utf-8')
    return pd.read_csv(StringIO(data))


def convert_to_sacks(data, field):
    data['peso_em_sacas'] = data[field] / 60
    return data


data = read_s3_data(bucket_name, file_key)


field_name = 'peso_em_kg'
data_transformed = convert_to_sacks(data, field_name)

print(data_transformed)
```

---
# 3- CI/CD e DevOps

Sua equipe usa o Airflow para gerenciar pipelines de dados. Você precisa configurar um pipeline CI/CD que:

Valide mudanças no código do Airflow antes do deploy.

Realize testes automatizados em DAGs (Direcioned Acyclic Graphs).

Envie alertas para o time se algo falhar durante o deploy.

## Tarefas:

Descreva o fluxo do pipeline CI/CD e ferramentas utilizadas.

Crie um arquivo YAML básico para uma ferramenta CI/CD (ex.: GitHub Actions ou GitLab CI).

Explique como garantir que o ambiente de produção esteja protegido contra deploys com falhas.

# Resposta
### Fluxo do Pipeline CI/CD

1. **Controle de Versão**:
   - **Ferramenta**: Git
   - **Descrição**: O código fonte do Airflow, incluindo DAGs e scripts auxiliares, é armazenado em um repositório Git. Isso permite controle de versão, colaboração e histórico de alterações.

2. **Validação de Código**:
   - **Ferramenta**: GitHub Actions
   - **Descrição**: Ao criar um novo branch ou fazer uma alteração no código, são executados jobs de validação que verificam a conformidade do código com padrões definidos (linting), verificam a sintaxe e aplicam outras validações estáticas.

3. **Testes Automatizados**:
   - **Ferramenta**: pytest
   - **Descrição**: As DAGs e scripts do Airflow são submetidos a testes automatizados para garantir que funcionem conforme esperado. Isso inclui testes unitários, testes de integração e testes de ponta a ponta.

4. **Build**:
   - **Ferramenta**: Docker
   - **Descrição**: Uma imagem Docker é construída contendo todas as dependências necessárias para o ambiente de execução do Airflow. Isso garante que o ambiente seja consistente em diferentes estágios de desenvolvimento e produção.

5. **Implantação Contínua**:
   - **Ferramenta**: AWS CodeDeploy
   - **Descrição**: As imagens Docker são implantadas em um ambiente de teste ou produção usando ferramentas de orquestração e implantação contínua. Isso garante que as mudanças no código sejam entregues de forma consistente e automatizada.

6. **Monitoramento e Alertas**:
   - **Ferramenta**: Prometheus
   - **Descrição**: O pipeline inclui monitoramento contínuo e configuração de alertas. Qualquer falha nos DAGs ou anomalias detectadas durante a execução acionam alertas para a equipe responsável, permitindo uma resposta rápida.

7. **Gerenciamento de Configuração**:
   - **Ferramenta**: Terraform
   - **Descrição**: As configurações de infraestrutura e do ambiente do Airflow são gerenciadas como código, garantindo que qualquer mudança seja rastreável e reprodutível. (Idempotente)
## Script YAML

```yaml
name: Airflow CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.8'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install apache-airflow
      - name: Run linting
        run: |
          pip install flake8
          flake8 dags/

  test:
    runs-on: ubuntu-latest
    needs: validate
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.8'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install apache-airflow
      - name: Run tests
        run: |
          pytest tests/

  deploy:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Deploy to production
        run: |
          ./deploy_script.sh
      - name: Notify team
        uses: slackapi/slack-github-action@v1.16.0
        with:
          channel-id: ${{ secrets.SLACK_CHANNEL_ID }}
          slack-message: 'Deploy executado com sucesso!!'
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

```
---

Para garantir que o ambiente de produção esteja protegido contra deploys com falhas, o YAML acima inclui etapas de validação e testes automatizados. Apenas se ambos forem bem-sucedidos, o deploy é realizado. O uso de um ambiente de staging ajuda a detectar problemas sem impactar a produção. As notificações via Slack informam a equipe sobre o estado do deploy. O monitoramento contínuo com ferramentas como Prometheus, juntamente com alertas em tempo real, ajuda a detectar e responder rapidamente a problemas. Essas práticas garantem a confiabilidade e a continuidade dos serviços em produção.

---
# 4- Modelagem de Dados
Um cliente deseja uma base de dados para gerenciar pedidos, clientes e produtos de uma loja virtual. Ele precisa saber:

- Quais são os produtos mais vendidos
- O histórico de compras de cada cliente
- A evolução mensal do faturamento

## Tarefas:
- Modelar o banco de dados, criando um esquema ER (Entidade Relacionamento)
- Escrever scripts SQL para:
  - Retornar os produtos mais vendidos
  - Listar o histórico de compras de um cliente específico
  - Calcular o faturamento mensal
# Resposta
## Para resolver esta questão, criei um banco postgresql em minha máquina utilizando docker
Print utilizando o terminal linux do container para comunicar com o banco e me certificar que as tabelas foram criadas
![image](https://github.com/user-attachments/assets/11577f1d-8c1b-476c-9f99-1ed482187d2e)

Print tirado das tabelas criadas no DBeaver
![image](https://github.com/user-attachments/assets/b2fe4d2f-19b9-45aa-874b-d6841694ffa3)

Print dados adicionados nas tabelas **CLIENTE, PRODUTO, PEDIDO e PEDIDOPRODUTO**, respectivamente
![image](https://github.com/user-attachments/assets/bdf7c691-0571-4ecc-bc9b-3d7e9db46d48)

Abaixo temos as consultas:
## Script SQL
```sql
--Produtos mais vendidos
SELECT P.Nome, SUM(PP.Quantidade) AS TotalVendido
FROM PedidoProduto PP
JOIN Produto P ON PP.ProdutoID = P.IDProduto
GROUP BY P.Nome
ORDER BY TotalVendido DESC;

--Histórico de compras cliente especìfico
SELECT C.Nome, P.DataPedido, PR.Nome, PP.Quantidade, PP.PrecoUnitario
FROM PedidoProduto PP
JOIN Pedido P ON PP.PedidoID = P.PedidoID
JOIN Cliente C ON P.ClienteID = C.ClienteID
JOIN Produto PR ON PP.ProdutoID = PR.IDProduto
WHERE C.ClienteID = 1
ORDER BY P.DataPedido;

--Cálculo faturamento mensal
SELECT extract(year from P.DataPedido) AS Ano, extract (month from P.DataPedido) AS Mes, SUM(P.TotalPedido) AS FaturamentoMensal
FROM Pedido P
GROUP BY extract(year from P.DataPedido), extract (month from P.DataPedido) 
ORDER BY Ano, Mes;
```
Print resultado das 3 consultas, na ordem que aparecem no script acima:
![image](https://github.com/user-attachments/assets/16eb8a12-ad71-4932-b73e-621207405ce4)

