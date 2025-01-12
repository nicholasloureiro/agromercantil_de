Cenário de Banco de Dados (DBA)
Sua empresa possui um banco de dados PostgreSQL com uma tabela de logs que cresce em 1 milhão de linhas por dia. Atualmente, as consultas realizadas nessa tabela estão extremamente lentas.

Tarefa
Proponha uma solução que otimize a performance para consultas de logs recentes (últimos 7 dias) sem degradar a consulta de logs antigos.

Explique como você implementaria:

Particionamento de tabelas

Estratégia de indexação

Backup e manutenção regular da tabela.

Resposta
Minha primeira ação seria perguntar se há necessidade de manter esses logs no banco de dados e qual a frequência de acesso a esses dados, de acordo com o período. Por exemplo, se o acesso é frequente somente dentro do período de 7 dias, é interessante tirar os dados mais antigos do banco e movê-los para algum armazenamento preparado para acomodar uma massa de dados deste tamanho sem impactar a performance e os custos.

Caso fosse confirmado que os logs depois de 7 dias são raramente acessados, eu utilizaria o recurso de exportação de dados do PostgreSQL para exportar logs antigos periodicamente para um bucket S3 e faria a configuração das políticas de ciclo de vida para otimizar os custos. As políticas de ciclo de vida são os tipos de armazenamento dentro do S3; dados que são acessados com menos frequência são naturalmente mais baratos de armazenar. Porém, há algumas opções que possuem delay para apresentar os dados, por isso é importante confirmar se a frequência de acesso muda de acordo com a "idade" do log e qual a urgência de obter dados mais antigos, tendo em vista que há opções com delays de até 12 horas, e que quanto maior a espera, menor é o custo da opção.

Eu pensei em duas políticas de ciclo de vida interessantes para este caso:

1- S3 Glacier Instant Retrieval para manter os logs por 30 dias, com delay de acesso de milissegundos.

2- S3 Glacier Flexible Retrieval para manter os logs até 90 dias, com delay de acesso entre 1 minuto e 12 horas. Após 90 dias, os logs seriam excluídos.
