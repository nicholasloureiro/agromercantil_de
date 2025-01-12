1 - Cenário de Banco de Dados (DBA)
Sua empresa possui um banco de dados PostgreSQL com uma tabela de logs que cresce em 1 milhão
de linhas por dia. Atualmente, as consultas realizadas nessa tabela estão extremamente lentas.

Tarefa

 Proponha uma solução que otimize a performance para consultas de logs recentes (últimos 7 dias
sem degradar a consulta de logs antigos

 Explique como você implementaria
 Particionamento de tabelas
 Estratégia de indexação
 Backup e manutenção regular da tabela.

R: Minha primeira açao seria perguntar se ha necessidade de manter estes logs no banco de dados, e qual a frequencia de acesso a estes dados, de acordo com o periodo. Por exemplo, se o 
acesso e frequente somente dentro do periodo de 7 dias, e interessante tirar os dados mais antigos do banco e move-los para algum armazenamento preparado para acomodar uma massa de dados 
deste tamanho. Caso fosse confirmado que os logs depois de 7 dias sao RARAMENTE utilizados, eu utilizaria o recurso de exportação de dados do PostgreSQL para exportar logs antigos periodi-
camente para um bucket S3, e faria a configuraçao das politicas de ciclo de vida para otimizar os custos (As politicas de ciclo de vida sao os tipos de armazenamento dentro do S3, dados 
que sao acessados com menos frequencia, naturalmente, sao mais baratos de armazenar, porem ha algumas opçoes que apresentam delay para apresentar os dados, por isso e importante confirmar
se a frequencia de acesso muda de acordo com a "idade" do log, e qual a urgencia de obter dados mais antigos, tendo em vista que ha opçoes com delays de ate 12 horas, e que quanto maior a
espera, menor e o custo da opçao)

![image](https://github.com/user-attachments/assets/8c54e02b-135b-4155-90cb-dbbe3a67d7eb)





