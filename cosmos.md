Consultas do AzureDiagnostics
Se o modo herdado for escolhido, os dados de diagnóstico serão armazenados na tabela AzureDiagnostics, portanto, todas as consultas kusto serão executadas nessa tabela. Como vários recursos do Azure também podem estar preenchendo essa tabela, inclua o filtro ResourceProvider=="MICROSOFT.DOCUMENTDB" em sua cláusula where para retornar apenas as entradas do Azure Cosmos DB. Além disso, para diferenciar entre os diferentes logs que você escolheu nas configurações de diagnóstico, adicione um filtro à coluna Category. Por exemplo, para retornar documentos para o log QueryRuntimeStatistics, inclua a cláusula where| where ResourceProvider=="MICROSOFT.DOCUMENTDB" and Category=="QueryRuntimeStatistics". O Kusto diferencia maiúsculas de minúsculas, portanto, verifique isso nos nomes da coluna. Vamos examinar alguns Kustoexemplos de consulta usando a tabela AzureDiagnostics.

Consulta que retorna a contagem e a solicitação total cobrada dos diferentes tipos de operação do Azure Cosmos DB na última hora.


````
AzureDiagnostics 
| where TimeGenerated >= ago(1h)
| where ResourceProvider=="MICROSOFT.DOCUMENTDB" and Category=="DataPlaneRequests" 
| summarize OperationCount = count(), TotalRequestCharged=sum(todouble(requestCharge_s)) by OperationName
| order by TotalRequestCharged desc 
````
Crie uma consulta que retorne um gráfico de tempo para todas as solicitações bem-sucedidas (status 200) e limitadas por taxa (status 429) na última hora. As solicitações serão agregadas a cada dez minutos.


````
AzureDiagnostics 
| where TimeGenerated >= ago(1h)
| where ResourceProvider=="MICROSOFT.DOCUMENTDB" and Category=="DataPlaneRequests" 
| summarize requestcount=count() by statusCode_s, bin(TimeGenerated, 10m)
| render timechart 
````


Ao contrário das consultas AzureDiagnostic, as consultas específicas do recurso serão executadas nas diferentes tabelas que foram criadas para cada categoria de log escolhida na caixa de diálogo de configuração de diagnóstico. Para usar essas tabelas, prefixe os nomes de tabela na lista acima com a cadeia de caracteres CDB. Vamos examinar alguns exemplos.

Consulta que retorna a contagem e a solicitação total cobrada dos diferentes tipos de operação do Azure Cosmos DB na última hora.
````
CDBDataPlaneRequests
| where TimeGenerated >= ago(1h)
| summarize OperationCount = count(), TotalRequestCharged=sum(todouble(RequestCharge)) by OperationName
| order by TotalRequestCharged desc
````
Crie uma consulta que retorna um grafo de gráfico de tempo para todas as solicitações bem-sucedidas (status 200) e de taxa limitada (status 429) na última hora.


````
CDBDataPlaneRequests 
| where TimeGenerated >= ago(2h)
| summarize requestcount=count() by StatusCode, bin(TimeGenerated, 10m)
| render timechart 
````


Para determinar quais tipos de solicitação estão causando as exceções 429, a execução de uma consulta em Logs de Diagnóstico do Azure pode retornar as RUs consumidas por tipo de solicitação. A consulta do exemplo abaixo retorna a média de RUs por minuto por operação para essas operações com exceções 429.

````
AzureDiagnostics
| where TimeGenerated >= ago(24h)
| where Category == "DataPlaneRequests"
| summarize throttledOperations = dcountif(activityId_g, statusCode_s == 429), totalOperations = dcount(activityId_g), totalConsumedRUPerMinute = sum(todouble(requestCharge_s)) by databaseName_s, collectionName_s, OperationName, requestResourceType_s, bin(TimeGenerated, 1min)
| extend averageRUPerOperation = 1.0 * totalConsumedRUPerMinute / totalOperations 
| extend fractionOf429s = 1.0 * throttledOperations / totalOperations
| order by fractionOf429s desc
````

Algumas soluções possíveis para esse tipo de exceção 429 são:

Se for determinado que as exceções 429 ocorreram devido a uma partição quente, considere alterar a chave de partição.
Se as exceções não foram causadas por uma partição quente, aumentar as RU/s no contêiner poderá ser a solução.
Se as exceções ocorreram em solicitações de documento de consulta, solucione o problema das consultas com cobranças altas de RU.
