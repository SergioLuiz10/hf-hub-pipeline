hf-hub-pipeline

Pipeline de dados em arquitetura medalhão com PySpark e Delta Lake, processando metadados de modelos do Hugging Face Hub.

Dataset

Metadados dos modelos publicados no Hugging Face Hub, puxados da API pública. Cada modelo traz id, autor, downloads, likes, tags, licença, datas de criação e modificação.

O que descobri inspecionando o dado

A API responde rápido. Testei com limit de 5, 100 e 1000 modelos. Todos voltaram em menos de um segundo, mesmo trazendo o payload completo. A rede não é o gargalo aqui.

Vem createdAt e lastModified no JSON. Isso permite carga incremental: em vez de apagar e regravar tudo toda vez, dá pra mexer só no que mudou desde a última execução.

O campo tags mistura duas coisas. Tem rótulo solto como pytorch e bert, e tem par chave-valor como license:apache-2.0 e dataset:s2orc. Pra conseguir filtrar por licença, ela precisa sair de dentro da lista e virar coluna própria.

Nem toda tag vale virar coluna. A region:us aparece em todo modelo do Hub. Uma coluna com o mesmo valor em toda linha não responde pergunta nenhuma.

Fan-out. Um modelo tem uma licença, mas pode ter vinte datasets. Se cada par modelo-dataset virar uma linha, o mesmo modelo aparece vinte vezes na tabela. Aí somar downloads dá 254 milhões vezes vinte, mais de 5 bilhões. O número sai errado e o pipeline não acusa erro nenhum.

Resolvi separando em duas tabelas:

Tabela	Uma linha por	Serve pra
modelos	modelo	somar downloads, contar likes
modelo_dataset	modelo + dataset	contar quais datasets mais aparecem

Escopo. A API entrega mil por vez e o Hub tem centenas de milhares de modelos. Puxar tudo daria muita chamada encadeada e risco de estourar a cota diária do Free Edition. Vou de 10 a 20 mil, que já é volume suficiente pro Spark fazer sentido.




## Camada Bronze

**Fonte:** API pública do HuggingFace (`https://huggingface.co/api/models`)
**Volume:** 10.000 modelos, 10.000 ids únicos
**Tabela:** `bronze_modelos` (Delta, managed no Unity Catalog)

### Schema

| Coluna | Tipo | Origem |
|---|---|---|
| `_id` | string | API |
| `author` | string | API |
| `createdAt` | string | API |
| `downloads` | bigint | API |
| `gated` | string | API |
| `id` | string | API |
| `lastModified` | string | API |
| `library_name` | string | API |
| `likes` | bigint | API |
| `modelId` | string | API |
| `pipeline_tag` | string | API |
| `private` | boolean | API |
| `sha` | string | API |
| `siblings` | array&lt;map&lt;string,string&gt;&gt; | API |
| `tags` | array&lt;string&gt; | API |
| `data_ingestao` | timestamp | auditoria |
| `origem` | string | auditoria |

As 15 primeiras colunas vêm da API sem nenhuma alteração. As duas últimas
foram adicionadas na ingestão e respondem "de onde veio essa linha e quando
ela chegou".

### Por que nada foi transformado

A Bronze é cópia fiel da origem. Nenhuma coluna foi renomeada, nenhum tipo foi
convertido, nenhum nulo foi tratado. Limpeza é trabalho da Silver.

O motivo é prático: se daqui a três meses uma transformação da Silver estiver
errada, a Bronze ainda tem o dado original para reprocessar. Se a limpeza
acontecesse já na ingestão, o dado cru estaria perdido e o bug seria
irreversível.

Os tipos "tortos" no schema acima são intencionais, não descuido:

| Coluna | Como está | Como deveria ficar na Silver |
|---|---|---|
| `createdAt` | string | timestamp |
| `lastModified` | string | timestamp |
| `gated` | string | boolean (a API mistura `false`, `"auto"`, `"manual"`) |
| `siblings` | array de map | achatado ou descartado |
| `_id` / `id` / `modelId` | três identificadores | escolher um como chave |

Tudo isso fica para a próxima camada.

Stack

Databricks Free Edition (serverless) · PySpark · Delta Lake · Python
