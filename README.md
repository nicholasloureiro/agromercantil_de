# Cenário de Banco de Dados (DBA) #

Sua empresa possui um banco de dados PostgreSQL com uma tabela de logs que cresce em 1 milhão de linhas por dia. Atualmente, as consultas realizadas nessa tabela estão extremamente lentas.

# Tarefa #
Proponha uma solução que otimize a performance para consultas de logs recentes (últimos 7 dias) sem degradar a consulta de logs antigos.

Explique como você implementaria:

Particionamento de tabelas

Estratégia de indexação

Backup e manutenção regular da tabela.

# Resposta #

Seguindo as orientações da pergunta, eu até poderia particionar a tabela de logs por intervalo de data, criar índices baseados no ID do log ou na sua data de criação e realizar backups incrementais diários e completos semanais para garantir a integridade dos dados. Mas, a longo prazo, isto seria uma má solução para este problema (e já está sendo, pois foi dito que as consultas estão extremamente lentas), além do custo de manter uma massa de dados deste tamanho no banco. Se for no RDS, por exemplo:

## Estimativa de custo

Vou supor que cada log tenha um tamanho médio de 1 KB (1000 bytes).

Volume de Dados Mensal:

1 milhão de linhas/dia * 1 KB/linha * 30 dias = **30 GB/mês**

## Custo do RDS:

O custo do RDS depende da instância escolhida. Vou considerar uma instância db.t2.medium, que custa aproximadamente **R$ 0,15** por hora.

## Custo de Armazenamento:

O custo de armazenamento no RDS é de aproximadamente **R$ 0,02 por GB/mês.**

## Custo de Backup:

O custo de backup incremental é de aproximadamente **R$ 0,01 por GB/mês.**

## Total Estimado

Custo da Instância RDS: 24 horas/dia * 30 dias * R$ 0,15/hora = **R$ 108/mês**

Custo de Armazenamento: 30 GB * R$ 0,02/GB = **R$ 0,60/mês**

Custo de Backup: 30 GB * R$ 0,01/GB = **R$ 0,30/mês**

Total Estimado: R$ 108 + R$ 0,60 + R$ 0,30 = **R$ 108,90/mês**


--------------------------------------------------------------------------------


Minha primeira ação seria perguntar se há **necessidade** de manter esses logs no banco de dados e qual a frequência de acesso a esses dados, de acordo com o período. Por exemplo, se o acesso é frequente somente dentro do período de 7 dias, é interessante tirar os dados mais antigos do banco e movê-los para algum armazenamento preparado para acomodar uma massa de dados deste tamanho, melhorando a performance e os custos.

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


# REDUÇÃO DE CUSTOS DE APROXIMADAMENTE 98%

Ref: https://aws.amazon.com/pt/s3/pricing/?gclid=CjwKCAiA7Y28BhAnEiwAAdOJUHkHk7-4SguNMVbmbXqOCnowlx4VJ56qu33bwgQvq43Ra_tL1m5-ABoChmsQAvD_BwE&trk=3daf7be6-997a-4e7c-af79-be4074c210e4&sc_channel=ps&ef_id=CjwKCAiA7Y28BhAnEiwAAdOJUHkHk7-4SguNMVbmbXqOCnowlx4VJ56qu33bwgQvq43Ra_tL1m5-ABoChmsQAvD_BwE:G:s&s_kwcid=AL!4422!3!536456042828!p!!g!!amazon%20s3%20data%20storage%20pricing!12024810840!115492236145

Ref: https://aws.amazon.com/pt/rds/aurora/pricing/

----------------------------------------------------------------------------------------------------------------------------------------



