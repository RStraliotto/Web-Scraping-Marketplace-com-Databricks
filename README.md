# Web Scraping e Análise de Dados no Mercado Livre

Este repositório contém um pipeline de **Web Scraping** que coleta dados detalhados sobre notebooks de diversas marcas no Mercado Livre, utilizando Databricks para processamento e análise de dados.

## Descrição do Projeto

O projeto realiza scraping de páginas do Mercado Livre para extrair informações sobre notebooks de marcas como Dell, HP, Lenovo e Samsung. Os dados coletados incluem preços, avaliações, promoções e outras características dos produtos. Os dados são processados e salvos em um arquivo CSV no Databricks, onde são analisados e exibidos.

## Tecnologias Utilizadas

- **Python**: Para scraping de dados e manipulação com bibliotecas como `requests` e `BeautifulSoup`.
- **Pandas**: Para criação e manipulação de DataFrames.
- **Spark**: Para processamento de dados em grande escala e salvamento no DBFS (Databricks File System).
- **Databricks**: Ambiente para execução e visualização de notebooks e pipelines.

## Instruções de Uso

### Configuração do Ambiente

Certifique-se de ter uma conta no [Databricks](https://databricks.com) e um workspace configurado.

### Executando o Pipeline

1. **Criação do Notebook**: No Databricks, crie um novo notebook e selecione a linguagem Python.

2. **Inserção do Código**: Copie e cole o código abaixo no notebook Databricks.

    ```python
    # Databricks notebook source
    import requests
    from bs4 import BeautifulSoup
    from datetime import datetime
    from urllib.parse import urlparse
    from time import sleep
    import pytz
    import pandas as pd
    from pyspark.sql import SparkSession

    # Função para obter a última palavra da URL
    def get_last_word(url):
        parsed_url = urlparse(url)
        path = parsed_url.path.rstrip('/')
        return path.split('/')[-1]

    # Lista de URLs
    urls = [
        'https://lista.mercadolivre.com.br/informatica/portateis-acessorios/notebooks/dell',
        'https://lista.mercadolivre.com.br/informatica/portateis-acessorios/notebooks/hp',
        'https://lista.mercadolivre.com.br/informatica/portateis-acessorios/notebooks/lenovo',
        'https://lista.mercadolivre.com.br/informatica/portateis-acessorios/notebooks/samsung/',
    ]

    # Criar uma lista para armazenar todos os dados
    all_data = []

    # Cabeçalho do agente de usuário para simular um navegador
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.36'
    }

    # Fuso horário do Brasil
    brasil_timezone = pytz.timezone('America/Sao_Paulo')

    # Iterar sobre as URLs
    for url in urls:
        # Fazer a solicitação
        page = requests.get(url, headers=headers)
        soup = BeautifulSoup(page.text, 'html.parser')

        # Encontrar todos os elementos 'div' com a classe 'ui-search-result'
        divs = soup.find_all('div', class_='ui-search-result')

        # Adicionar coluna de data e hora da execução
        execution_time = datetime.now(brasil_timezone).strftime("%Y-%m-%d")

        # Adicionar coluna de sequência
        sequence = 1

        # Adicionar coluna de origem
        source = get_last_word(url)

        # Criar uma lista para armazenar os links dos produtos coletados nesta página
        current_page_links = []

        # Iterar sobre os elementos da página
        for element in divs:
            # Extrair o link do elemento
            link = element.find('a', class_='ui-search-link')['href']

            # Verificar se o link já foi coletado anteriormente
            if link not in current_page_links:
                # Adicionar o link à lista de links da página atual
                current_page_links.append(link)

                # Extrair dados do produto e adicionar à lista de dados
                img_src = element.find('img', class_='ui-search-result-image__element').get('src', '')
                if 'data:image' in img_src:
                    img_src = "https://http2.mlstatic.com/" + img_src.split(',')[1]

                product = {
                    "Data": execution_time,
                    "Posição_Pg": sequence,
                    "Origem": source,
                    "Titulo": element.find('h2', class_='ui-search-item__title').text if element.find('h2', class_='ui-search-item__title') else None,
                    "PrecoOriginal": element.find('span', class_='andes-money-amount__fraction').text if element.find('span', class_='andes-money-amount__fraction') else None,
                    "PrecoDesconto": element.find('span', class_='ui-search-price__part').text if element.find('span', class_='ui-search-price__part') else None,
                    "Desconto": element.find('span', class_='ui-search-price__second-line__label').text if element.find('span', class_='ui-search-price__second-line__label') else None,
                    "Parcelamento": element.find('span', class_='ui-search-installments').text if element.find('span', class_='ui-search-installments') else None,
                    "LinkElemento": link,
                    "Avaliacao": element.find('span', class_='ui-search-reviews__rating-number').text if element.find('span', class_='ui-search-reviews__rating-number') else None,
                    "QtdAvaliacoes": element.find('span', class_='ui-search-reviews__amount').text if element.find('span', class_='ui-search-reviews__amount') else None,
                    "Vendido": element.find('p', class_='ui-search-official-store-label').text if element.find('p', class_='ui-search-official-store-label') else None,
                    "Patrocinado": element.find('label', class_='ui-search-item__pub-label').text if element.find('label', class_='ui-search-item__pub-label') else None,
                    "Frete": element.find('span', class_='ui-pb-highlight').text if element.find('span', class_='ui-pb-highlight') else None,
                    "EnvioFull": element.find('p', class_='fulfillment').text if element.find('p', class_='fulfillment') else None,
                    "ImgURL": img_src,
                }
                all_data.append(product)
                sequence += 1

        # Aguarde alguns segundos para evitar sobrecarregar o servidor
        sleep(5)

    # Nome do arquivo CSV com o nome do processo e a data
    process_name = "meli"
    date_suffix = datetime.now(brasil_timezone).strftime("%Y%m%d")
    csv_file_path = f'dbfs:/FileStore/landing_zone/Market/{process_name}_{date_suffix}.csv'

    # Criar um DataFrame do Pandas a partir da lista de dicionários
    df = pd.DataFrame(all_data)

    # Remover os sinais de menos da coluna "QtdAvaliacoes"
    df['QtdAvaliacoes'] = df['QtdAvaliacoes'].str.replace('-', '')

    # Salvar os dados em um novo arquivo CSV no DBFS usando Spark
    spark = SparkSession.builder.getOrCreate()
    df_spark = spark.createDataFrame(df)
    df_spark.write.format("csv").option("header", "true").mode("overwrite").save(csv_file_path)

    print(f"Dados salvos em: {csv_file_path}")

    # COMMAND ----------

    from pyspark.sql import SparkSession

    # Cria uma SparkSession
    spark = SparkSession.builder.appName("List CSV Data").getOrCreate()

    # Define o caminho do arquivo CSV
    csv_file_path = "dbfs:/FileStore/landing_zone/Market/meli_20240804.csv"

    # Lê o arquivo CSV usando Spark
    df = spark.read.csv(csv_file_path, header=True, inferSchema=True)

    # Exibe o esquema do DataFrame
    df.printSchema()

    # Exibe as primeiras linhas do DataFrame com uma visualização mais limpa
    #df.show(truncate=False)

    # Para visualização tabular mais clara e customizada, você pode converter o DataFrame para Pandas e usar a função `display` do Databricks
    pdf = df.toPandas()
    import pandas as pd

    # Exibe o DataFrame como uma tabela no notebook
    display(pd.DataFrame(pdf))
    ```

3. **Executando o Código**: Execute as células do notebook em ordem para coletar os dados e visualizá-los.

## Contribuição

Sinta-se à vontade para contribuir com melhorias ou correções para o projeto. Por favor, siga as diretrizes padrão para pull requests e commits.

## Licença

Este projeto está licenciado sob a [Licença MIT](LICENSE).

## Contato

Para mais informações ou suporte, entre em contato com [romulostraliotto@gmail.com](mailto:seu-email@exemplo.com).

