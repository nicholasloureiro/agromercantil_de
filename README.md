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

Adicionaria novos elementos ao template do KDG e enviaria os dados ao Firehose, validando a ingestão no S3 e atualizando as definições de tabela com o Glue Crawler. Consultaria a tabela no Redshift Spectrum para ler os dados combinados de dois schemas. Rodaria o script python que se encontra no final da resposta desta questão, que cria uma coluna nova na estrutura, enviaria ao Firehose, validaria a ingestão no S3 e atualizaria as definições com o Glue Crawler.

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
### Executando o script
- Código teste feito apenas para demonstrar a etapa de leitura do S3 e aplicação de uma transformação simples.
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

Ref: KLEPPMANN, M. Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems. São Francisco: O'Reilly Media, 2017.

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

---
# 5- Web Scraping

Você precisa monitorar os preços de produtos em um site de e-commerce que atualiza o layout com frequência e bloqueia bots de forma ativa.

## Tarefas:

- Explique como configurar um scraper resiliente para evitar bloqueios.
- Escreva um código básico que realize o scraping de uma página fictícia com Python.
- Detecte mudanças no layout (ex.: mudanças de tags HTML).

# Resposta

Antes de fazer o scraping, é interessante inspecionarmos a página e olharmos o arquivo robots.txt do site que iremos raspar dados, é um arquivo de texto simples que os administradores de sites utilizam para direcionar os bots dos motores de busca sobre como devem acessar suas páginas, é importante que nosso código não quebre as regras deste arquivo. Neste exemplo vou realizar o scraping na página inicial do Magazine Luiza, normalmente eu utilizaria somente as bibliotecas Requests e BeautifulSoup (se dão muito bem com HTML estático), mas como o site do Magazine Luiza é carregado dinâmicamente com JavaScript, foi necessário utilizar a biblioteca Selenium para acessar a página e BeautifulSoup para analisar o HTML (agora sim, estático). Cada vez que o código roda, ele gera um hash do conteúdo da página e compara com o hash anterior, se os hashes forem diferentes, significa que o layout mudou. Além disso também salvo a pagina em HTML estático para análises posteriores.
Para evitar que o site detecte e bloqueie o scraper, usei a biblioteca random para adicionar variação nos tempos de espera e nos user agents, tentando simular um comportamento mais humano. Essa aleatoriedade aliada ao código, que não fere as regras do arquivo robots.txt ajuda a tornar o scraper mais difícil de ser detectado e bloqueado pelo site.

### Robots.txt do Magalu: https://www.magazineluiza.com.br/robots.txt
### URL utilizada: https://www.magazineluiza.com.br/

## Script Python

```python

from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
from bs4 import BeautifulSoup
from datetime import datetime
import hashlib
import random
import time
import os
import re
import requests
from unittest.mock import patch, MagicMock


url = 'https://www.magazineluiza.com.br/'


user_agents = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36',
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Firefox/89.0 Safari/537.36',
    
]


options = webdriver.ChromeOptions()
options.add_argument('--headless')  
driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)


def get_page_hash(url):
    headers = {'User-Agent': random.choice(user_agents)}
    driver.get(url)
    time.sleep(random.uniform(3, 7))  
    soup = BeautifulSoup(driver.page_source, 'html.parser')
    page = requests.get(url, headers=headers) 
    time_of_run = datetime.now().strftime("%Y_%m_%d-%I_%M_%S_%p")
    with open(f'magalu_html_{time_of_run}.txt', 'wb+') as f: 
        f.write(page.content)
    return hashlib.md5(soup.prettify().encode('utf-8')).hexdigest()


def monitor_changes(url):
    previous_hash = None
    while True:
        try:
            current_hash = get_page_hash(url)
            if previous_hash and current_hash != previous_hash:
                print("Mudança detectada no layout!")
            else:
                print("Nenhuma mudança detectada.")
            previous_hash = current_hash
        except Exception as e:
            print(f"Erro ao acessar a página: {e}")
        time.sleep(random.uniform(270, 330))  


monitor_changes(url)


driver.quit()


# Testes
def test_get_page_hash(): 
    url = 'https://www.magazineluiza.com.br/' 
    with patch('random.choice', return_value=user_agents[0]), patch('selenium.webdriver.Chrome.get'), patch('time.sleep'): 
        hash_value = get_page_hash(url)
        assert re.match(r"^[a-fA-F0-9]{32}$", hash_value), "O valor hash retornado é inválido"


def test_monitor_changes(capsys):
    url = 'https://www.magazineluiza.com.br/'
    with patch('random.choice', return_value=user_agents[0]), patch('selenium.webdriver.Chrome.get'), patch('time.sleep', side_effect=[1, 1]):
        mock_get_page_hash = MagicMock(side_effect=['hash1', 'hash2'])
        with patch('__main__.get_page_hash', mock_get_page_hash):
            monitor_changes(url)
            assert mock_get_page_hash.call_count == 2, "A função get_page_hash não foi chamada corretamente"
            captured = capsys.readouterr()
            assert 'Mudança detectada no layout!' in captured.out, "Mudança no layout não detectada"


```
### Executando o Script:
- Instale as dependências: `pip install selenium webdriver-manager beautifulsoup4 requests`.
- Verifique se o Chrome está instalado para `webdriver_manager`.
- Execute o script, e ele monitorará o site para mudanças. Use `Ctrl+C` para parar o monitoramento (Não irá funcionar na rede da Agromercantil).
- Para executar os testes: `python3 -m pytest nome_do_arquivo.py`.
---

# 6- AWS e Infrastrutura

Sua empresa precisa processar grandes volumes de dados em tempo real e armazená-los para consultas futuras.

## Tarefas:
- Descreva uma arquitetura AWS que inclua:
  - Ingestão em tempo real (ex.: Kinesis).
  - Processamento (ex.: Lambda ou Glue).
  - Armazenamento (ex.: S3 e Redshift).
- Explique como dimensionar essa arquitetura para lidar com picos de tráfego.

# Resposta

## Neste exemplo vou utilizar como base a arquitetura da questão 2

![image](https://github.com/user-attachments/assets/fdbf6dbd-d227-4eb7-a61f-3d56e0dbdb5e)

No diagrama acima, todos os dados coletados dos sensores IoT são centralizados no Amazon S3. Utilizando a funcionalidade de evolução de schema do AWS Glue, o Amazon Redshift Spectrum pode lidar automaticamente com mudanças no schema dos dados, como a adição ou remoção de colunas (por mais que haja alterações de estrutura, o pipeline não quebraria). Isso é alcançado através de um crawler do AWS Glue, que lê e adapta as mudanças de schema com base nas estruturas dos arquivos no S3. O crawler cria um schema híbrido, compatível com datasets antigos e novos. Assim, todos os arquivos de dados ingeridos podem ser lidos de um local especificado no S3 por meio de uma única tabela do Amazon Redshift Spectrum, referenciando o catálogo de metadados do AWS Glue.

Dessa forma, conseguiria lidar com as mudanças no esquema sem quebrar o pipeline. Essa arquitetura jánestaria preparada para lidar com horários de pico, por si só somente pela escolha de serviços utilizados, pois Amazon Kinesis e AWS Glue escalam AUTOMATICAMENTE conforme a demanda. O Amazon S3 é altamente escalável e o Redshift pode ser dimensionado adicionando mais nós ao cluster ou aumentando a capacidade dos nós existentes, garantindo que os dados continuem fluindo mesmo com tráfego intenso.

---
# 7- Monitoramento e Alertas
Um pipeline de dados apresenta falhas esporádicas que só são descobertas dias depois.

## Tarefas:
- Proponha uma solução para implementar monitoramento e alertas em tempo real.
- Explique quais métricas monitorar (ex.: tempo de execução, volume de dados) e como configurar alertas no CloudWatch ou uma ferramenta equivalente.
# Resposta

Para resolver falhas no pipeline de dados, seria importante saber o tipo de pipeline, entender a linhagem dos dados, e em qual etapa do pipeline a falha está acontecendo, primeiro monitoraria o tempo de execução, volume de dados, falhas nas tarefas específicas e uso de recursos como CPU e memória. Usaria AWS CloudWatch para configurar essas métricas e criar dashboards em tempo real. Alarmes me avisariam se algo saísse do normal, por exemplo, se o tempo de execução estiver muito longo ou se o volume de dados estiver estranho. Para notificações, configuraria o Amazon SNS para enviar avisos por email, SMS ou push notifications. 

Como o enunciado não menciona o tipo de pipeline, isso dificulta a especificação de métricas e alertas mais precisos, porém com as seguintes práticas é possível descobrir onde ocorreram e como ocorreram os erros, acelerando a resolução:

- **Documentação Completa**: Sempre documentar todas as etapas do pipeline e as métricas a serem monitoradas. 
- **Testes Automatizados**: Implementar testes para validar cada componente do pipeline. Assim, descobrindo qual etapa está o erro.
- **Versionamento de Dados**: Utilizar versionamento para rastrear mudanças nos dados e no código do pipeline. Possibilitando um possível rollback.
- **Gerenciamento de Erros**: Definir estratégias claras para lidar com falhas e exceções.

# 8- Data Governance e Segurança

Sua empresa precisa garantir que os dados sensíveis armazenados em seu ambiente AWS estejam protegidos e em conformidade com regulamentações como LGPD ou GDPR.

## Tarefas:
- Proponha uma estratégia para garantir:
  - Controle de acesso apropriado em S3 e Redshift.
  - Criptografia de dados em repouso e em trânsito.
  - Logs detalhados de acesso e auditoria.
- Explique como você verificaria se a empresa está em conformidade com regulamentações de proteção de dados.
# Resposta

#### 1. Controle de Acesso Apropriado em S3 e Redshift
Para garantir o controle de acesso apropriado em S3 e Redshift, eu utilizaria o **Amazon Identity and Access Management (IAM)** para definir políticas de acesso detalhadas. Além disso, utilizaria **Amazon S3 Bucket Policies** e **Redshift IAM Policies** para controlar quem pode acessar e modificar os dados. Por fim, usaria o S3 Access Points para simplificar o gerenciamento de acesso a dados para o conjunto de aplicações nos conjuntos de dados compartilhados no S3. Não precisaria mais lidar com uma política de bucket única e complexa, com centenas de regras de permissão diferentes que precisam ser gravadas, lidas, rastreadas e auditadas. Com os Pontos de Acesso S3, poderia criar pontos de acesso específicos para cada aplicação, permitindo o acesso a conjuntos de dados compartilhados com políticas personalizadas.

Para conjuntos de dados compartilhados grandes, eu poderia decompor uma política de bucket grande em políticas de ponto de acesso discretas e separadas para cada aplicação que precisasse acessar o conjunto de dados compartilhados. Isso permitiria focar na criação da política de acesso correta para uma aplicação, sem se preocupar em interromper o que qualquer outra aplicação está fazendo no conjunto de dados compartilhados. Também poderia copiar dados com segurança em altas velocidades entre Pontos de Acesso da mesma região usando a API de cópia S3, aproveitando as redes internas da AWS e VPCs.

Um Ponto de Acesso do S3 poderia limitar todo o acesso ao armazenamento do S3 a partir de uma Virtual Private Cloud (VPC). Poderia também criar uma Política de Controle de Serviço (SCP) e exigir que todos os pontos de acesso fossem restritos a uma VPC, protegendo meus dados com firewall em redes privadas. Com os pontos de acesso, poderia testar facilmente novas políticas de controle de acesso antes de migrar aplicações para o ponto de acesso ou copiar a política para um ponto de acesso existente. O S3 Access Points permitiria especificar políticas de VPC endpoint que permitissem o acesso apenas a pontos de acesso (e, portanto, buckets) pertencentes a IDs de conta específicos. Isso simplificaria a criação de políticas de acesso que permitissem o acesso a buckets na mesma conta, enquanto rejeitaria qualquer outro acesso do S3 por meio do VPC endpoint.

#### 2. Criptografia de Dados em Repouso e em Trânsito
Para a criptografia de dados em repouso, utilizaria **Amazon S3 Server-Side Encryption (SSE)** e para dados em trânsito, utilizaria **TLS/SSL** para garantir a segurança durante a transferência de dados.

#### 3. Logs Detalhados de Acesso e Auditoria
Implementaria **Amazon CloudTrail** para registrar e monitorar todas as ações realizadas nos recursos da AWS. Além disso, utilizaria **Amazon S3 Access Logs** para registrar todas as solicitações de acesso aos buckets S3.

#### 4. Verificação de Conformidade com LGPD
Para verificar se a empresa está em conformidade com a LGPD, é necessário implementar uma matriz de impacto de riscos e garantir que todas as práticas de coleta, tratamento e armazenamento de dados pessoais estejam em conformidade com a legislação. Isso inclui obter consentimento explícito dos titulares dos dados, garantir a transparência nas operações de dados e implementar medidas de segurança adequadas.

### Exemplo Matriz de Risco
![image](https://github.com/user-attachments/assets/85384f1e-9eaa-4fd7-8c55-697347515cac)


Ref: https://certificacaoiso.com.br/matriz-de-riscos-o-que-e-e-como-aplicar/

Ref: https://aws.amazon.com/pt/blogs/

Ref: https://aws.amazon.com/pt/s3/features/access-points/

---

## 9- Data Lakes e Data Warehouses

Sua empresa está implementando um Data Lake em S3 para armazenar dados brutos e um Data Warehouse em Redshift para análises estruturadas.

## Tarefas
- Explique como você estruturaria os dados no Data Lake para facilitar o consumo no Data Warehouse.
- Proponha uma política de gerenciamento do ciclo de vida dos dados no Data Lake, considerando arquivamento e exclusão de dados antigos.
- Escreva um script de exemplo que integre dados do Data Lake ao Data Warehouse usando Python e SQL.

# Resposta

### Estruturação de Dados no Data Lake para Consumo no Data Warehouse

1. **Camadas do Data Lake**:
   - **Raw Layer (Camada Bruta)**:
     - Dados armazenados no formato original, sem transformações.
     - Organização por fonte de dados, data de ingestão e tipo (JSON, CSV, Parquet).
     - Exemplo de estrutura:  
       ```
       /raw/{fonte}/{ano}/{mes}/{dia}/{arquivo}
       /raw/erp/2025/01/13/sales_data.json
       ```
   - **Cleansed Layer (Camada Tratada)**:
     - Dados processados e padronizados (e.g., remoção de duplicatas, normalização).
     - Estrutura hierárquica por domínio de negócio e granularidade.
     - Exemplo:  
       ```
       /cleansed/{dominio}/{tabela}/{ano}/{mes}/{arquivo}
       /cleansed/vendas/faturamento/2025/01/faturamento.parquet
       ```
   - **Curated Layer (Camada Curada)**:
     - Dados organizados para atender consultas específicas do Data Warehouse.
     - Usaria formatos otimizados para leitura (Parquet) e particionamento eficiente.
     - Exemplo:
       ```
       /curated/{modelo_dimensional}/{dim_ou_fato}/{ano}/{mes}/{arquivo}
       /curated/star_schema/dim_cliente/2025/01/dim_cliente.parquet
       ```

2. **Organização e Catalogação**:
   - Utilizaria um **data catalog** (AWS Glue) para documentar as tabelas, metadados e regras de acesso.
   - Definiria **chaves de particionamento** relevantes (por data, região, cliente).

3. **Integração com o Data Warehouse**:
   - Criaria pipelines ETL/ELT para extrair dados da camada curada.
   - Utilizaria ferramentas de orquestração (Apache Airflow) para gerenciar cargas incrementais e completas.

---

### Política de Gerenciamento do Ciclo de Vida dos Dados no Data Lake

1. **Definição de Regras por Camada**: (Essa solução se parece com a arquitetura medalhão, utlizada pela Databricks)
   - **Raw Layer**:
     - Retenção de curto prazo (30-90 dias).
     - Arquivamento automático para armazenamento de baixo custo (Amazon S3 Glacier).
   - **Cleansed Layer**:
     - Retenção intermediária (6-12 meses).
     - Dados usados frequentemente permanecem ativos; dados obsoletos são arquivados.
   - **Curated Layer**:
     - Retenção de longo prazo (3-5 anos ou conforme regulamentações).
     - Dados antigos podem ser compactados ou transferidos para camadas menos acessíveis.
    
       ### Exemplo da arquitetura medalhão utilizada na Databricks
   ![image](https://github.com/user-attachments/assets/3b97ea94-e91c-4a27-b4f8-6325fd5dcf34)


2. **Automatização do Ciclo de Vida**:
   - Configuraria políticas de **lifecycle management** nas ferramentas de armazenamento (Amazon S3 Lifecycle).
   - Exemplo de política:
     - Dados não acessados há 90 dias na camada raw são movidos para arquivamento.
     - Dados arquivados há mais de 3 anos são excluídos.

3. **Backup e Compliance**:
   - Implementar backups automáticos regulares para atender requisitos legais.
   - Monitorar o acesso e auditoria dos dados com logs (AWS CloudTrail).

4. **Monitoramento e Otimização**:
   - Usar ferramentas de monitoramento (CloudWatch) para rastrear uso e custo.
   - Ajustar as políticas de retenção com base em padrões de acesso e custo-benefício.

5. **Economia com Políticas de Lifecycle**:
   - Ao mover dados da camada raw para o Amazon S3 Glacier após 90 dias, seria possível economizar significativamente, já que o S3 Glacier é uma opção de armazenamento de baixo custo.
   - Considerando que o custo do Amazon S3 Standard é de aproximadamente $0.023 por GB por mês, e o S3 Glacier é de cerca de $0.004 por GB por mês, a economia seria:
     ```
     Economia por GB por mês = $0.023 - $0.004 = $0.019
     ```
   - Se arquivarmos 1 TB de dados (1024 GB) na camada raw após 90 dias, nossa economia mensal seria:
     ```
     Economia mensal = 1024 GB * $0.019 * 6,03 = R$117,34
     ```

Essa abordagem não apenas organiza os dados para garantir fácil consumo no Data Warehouse, mas também mantém a eficiência do Data Lake ao longo do tempo, proporcionando economias significativas.

## Script Python
```python
import boto3
import psycopg2
from botocore.exceptions import NoCredentialsError


s3_bucket = 'data-lake-bucket'
s3_key = 'ano=2025/mes=01/dia=01/dados.csv'
redshift_host = 'redshift-cluster-url'
redshift_user = 'username'
redshift_password = 'password'
redshift_db = 'database'
redshift_table = 'data_table'
iam_role = 'arn:aws:iam::account-id:role/RedshiftCopyRole'

# Conectar ao S3
s3_client = boto3.client('s3')
try:
    s3_client.download_file(s3_bucket, s3_key, '/tmp/dados.csv')
    print("Arquivo baixado com sucesso do S3")
except NoCredentialsError:
    print("Credenciais AWS não encontradas")

# Conectar ao Redshift
try:
    conn = psycopg2.connect(
        dbname=redshift_db,
        user=redshift_user,
        password=redshift_password,
        host=redshift_host
    )
    cursor = conn.cursor()
    print("Conectado ao Redshift")

    # Copiar dados do S3 para o Redshift
    copy_query = f"""
    COPY {redshift_table}
    FROM 's3://{s3_bucket}/{s3_key}'
    IAM_ROLE '{iam_role}'
    CSV
    IGNOREHEADER 1;
    """
    cursor.execute(copy_query)
    conn.commit()
    print("Dados copiados para o Redshift")

    # Verificar dados copiados
    cursor.execute(f"SELECT * FROM {redshift_table} LIMIT 10")
    for row in cursor.fetchall():
        print(row)

  
    cursor.close()
    conn.close()
except Exception as e:
    print(f"Erro ao conectar ao Redshift: {e}")

```
### Executando o script
- Código teste feito apenas para demonstrar a integração de dados do Data Lake ao Data Warehouse.

Ref:https://docs.aws.amazon.com/redshift/

Ref: https://aws.amazon.com/pt/s3/pricing/?gclid=CjwKCAiA7Y28BhAnEiwAAdOJUHkHk7-4SguNMVbmbXqOCnowlx4VJ56qu33bwgQvq43Ra_tL1m5-ABoChmsQAvD_BwE&trk=3daf7be6-997a-4e7c-af79-be4074c210e4&sc_channel=ps&ef_id=CjwKCAiA7Y28BhAnEiwAAdOJUHkHk7-4SguNMVbmbXqOCnowlx4VJ56qu33bwgQvq43Ra_tL1m5-ABoChmsQAvD_BwE:G:s&s_kwcid=AL!4422!3!536456042828!p!!g!!amazon%20s3%20data%20storage%20pricing!12024810840!115492236145

Ref: https://www.databricks.com/br/glossary/medallion-architecture



  





