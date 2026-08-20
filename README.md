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

Stack

Databricks Free Edition (serverless) · PySpark · Delta Lake · Python
