# 📃 Full-text search

<p align="center">
  <a href="#goal">Goal</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#flux">Flux</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#escope">Escope</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#reprocessar">Reprocessar</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#recursos">Recursos</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#extras">Extras</a>&nbsp;&nbsp;&nbsp;
</p>

---

# Goal
### Allow users to enter a term in advanced search and receive results from contract snippets that contain the search term.
</br>

---
# Fluxo
## 1 - Collect from Google Cloud Storage the JSON files already processed by OCR;;

## 2 - Remove unnecessary fields from JSON files in order to make the file cleaner and indexing in Elasticsearch more performative;

## 3 - Index files in Elasticsearch;
---
# Escope

## Decide the indexer approach
### In the present context, the indexer is nothing more than responsible for receiving a JSON file with the content of the contract already extracted by [OCR](https://cloud.google.com/vision?utm_source=google&utm_medium=cpc&utm_campaign=latam-BR-all-pt-dr-BKWS-all-all-trial-e-dr-1011454-LUAC0014873&utm_content=text-ad-none-any-DEV_c-CRE_547331812249-ADGP_Hybrid%20%7C%20BKWS%20-%20EXA%20%7C%20Txt%20~%20AI%20%26%20ML_Vision-AI-KWID_43700066537016951-kwd-475108777409&utm_term=KW_google%20cloud%20vision-ST_Google%20Cloud%20Vision&gclid=CjwKCAjwrqqSBhBbEiwAlQeqGkzZwcTQX6IGQG3PtAyaLclzXGmKXh0CVC1PXSjv6LeNQ7gdbao4ohoCF40QAvD_BwE&gclsrc=aw.ds), performing the processing by removing or adding fields in the JSON file and finally performing the indexing in Elasticsearch.
</br>

### The chosen strategy was to create and configure a pipeline [Logstash](https://www.elastic.co/en/logstash/) to connect Google Cloud Storage and Elasticsearch.
</br>

## Configure or create an indexer
## 1 - Create a new logstash pipeline
</br>

### Create a file called contractContentFileSync.conf in <b>search/k8s/base/logstash</b>
```html
input {

}

filter {  
  
}

output {

}

```
### <b>A logstash pipeline is always composed of an input, a filter and an output.</b>
</br>


## 2 - Integrate the pipeline with Google Cloud Storage in order to collect the content of the contracts processed by the OCR.
</br>

### To perform this integration we use the plugin [logstash-input-google_cloud_storage](https://github.com/logstash-plugins/logstash-input-google_cloud_storage). With this plugin it is possible to authenticate with Google Cloud Storage, choose which bucket to connect to, make calls in dynamic time intervals to check if new files have entered the bucket, configure via regex the specific location of the necessary files and place metadata in each file in order to avoid reprocessing already downloaded files.

</br>

### The codec indicates that the input will be a JSON file
```html
input {
  google_cloud_storage {
    interval => 5  
    delete => false 
    bucket_id => "$${GOOGLE_STORAGE_BUCKET_PRIVATE}"
    json_key_file => "$${GOOGLE_APPLICATION_CREDENTIALS}"   
    file_matches => "organizations\\/.+\\/contracts\\/.+\\/files\\/.+\\/GoogleVision\\/.+\\.json"
    codec => "json"
  }  
}
```

## 3 - Remover campos desnecessários dos arquivos JSON e reestruturá-lo a fim de obter um [JSON Flat](https://www.ibm.com/docs/en/taw/1.3?topic=analytics-flat-json-versus-structured-json)
</br>

```html
filter {  
  json {
    source => "[message]"
  }

  mutate{
    remove_field => ["inputConfig"]
  }

  grok {
    match => {"[responses][0][context][uri]" => "(?<fileId>(?<=files\/)[\w]{8}-[\w]{4}-[\w]{4}-[\w]{4}-[\w]{12})"}
  }

  ruby{
    code => "
      responses = event.get('responses')
      if !responses.nil?
        if responses.length > 1 && !responses.each.nil?
          responses.each { |response|
            if !response['fullTextAnnotation'].nil?
              response['fullTextAnnotation'].delete('pages')
            end
          }  
        else
           if !responses[0]['fullTextAnnotation'].nil?
            responses[0]['fullTextAnnotation'].delete('pages')
          end
        end    
      end  
      event.set('responses', responses)
    "
  }

  split {
    field => "responses"
  }

  mutate {
    add_field => {
      "text" => "%{[responses][fullTextAnnotation][text]}"
      "uri" => "%{[responses][context][uri]}"
      "pageNumber" => "%{[responses][context][pageNumber]}"
    }
  }

  mutate {
    remove_field => ["responses"]
  }
}
```
### 3.1 Primeiramente indicaremos para o logstash que queremos filtrar um arquivo JSON utilizando o plugin [JSON Filter](https://www.elastic.co/guide/en/logstash/current/plugins-filters-json.html) built-in. O campo message passado para o source é um campo padrão do logstash, portanto para ter acesso ao conteúdo do JSON de fato precisamos indicá-lo no source. 
</br>

### Formato inicial do array
```
{
  inputConfig: {
    ...
  },
  "responses": [
    {
      "context": {
        "pageNumber": 1,
        "uri": "gs://private.linte.io/..."
      }
    },
    {
      "fullTextAnnotation": {
        "text": "text here",
        "pages": [
          ...
        ]
      }
    }
  ]    
}
```

### 3.2 Em seguida removeremos o campo inputConfig(um campo sem subfields)
</br>

### JSON após a remoção do inputConfig
```
{
  "responses": [
    {
      "context": {
        "pageNumber": 1,
        "uri": "gs://private.linte.io/..."
      }
    },
    {
      "fullTextAnnotation": {
        "text": "text here",
        "pages": [
          ...
        ]
      }
    }
  ]    
}
```

### 3.3 O [grok](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html) é um plugin built-in que permite passar um padrão e obter um match. No caso precisamos obter o id do arquivo, porém o mais próximo dessa informação que temos é apenas a url do arquivo no bucket com o seguinte padrão:

### <b>gs://private.linte.io/organizations/4d644eab-01d2-4532-a6d4-7bf48e05f8b7/contracts/d27cbac1-4396-4662-a20a-62298dc0ad1e/files/<mark>d8670f2c-443f-4943-9100-647ee0b4c21f</mark>.pdf</b>

### O uuid em amarelo é o id do arquivo contido em um array chamado responses, dentro de um objeto chamado context com uma chave chamada uri. No grok estamos indicando para o logstash criar um novo campo chamado fileId e que a regex deve capturar o uuid em amarelo.
</br>

### JSON após adicionar o campo fileId
```
{
  "responses": [
    {
      "context": {
        "pageNumber": 1,
        "uri": "gs://private.linte.io/..."
      }
    },
    {
      "fullTextAnnotation": {
        "text": "text here",
        "pages": [
          ...
        ]
      }
    }
  ],
  "fileId": "d8670f2c-443f-4943-9100-647ee0b4c21f"    
}
```

### 3.4 Devido a um array chamado pages contido dentro dos arquivos JSON a indexação tornava-se um processo pouco performático. O array de pages encontrava-se em responses => fullTextAnnotation => pages. Para removê-lo foi necessário utilizar um o plugin [ruby filter](https://www.elastic.co/guide/en/logstash/current/plugins-filters-ruby.html) built-in. No código ruby capturamos o campo responses utilizando a [API de  eventos](https://www.elastic.co/guide/en/logstash/current/event-api.html) do Logstash verificamos a existência do array de responses, e cobrimos três condições, a primeira envolve a possibilidade de existir mais de uma página no contrato, a segunda caso o contrato tenha páginas em branco. A última em decorrência de contratos que contêm apenas uma página com texto. 
</br>

### JSON após remover o array de pages
```
{
  "responses": [
    {
      "context": {
        "pageNumber": 1,
        "uri": "gs://private.linte.io/..."
      }
    },
    {
      "fullTextAnnotation": {
        "text": "text here",
      }
    }
  ],
  "fileId": "d8670f2c-443f-4943-9100-647ee0b4c21f"    
}
```

### 3.5 Para separar cada página do contrato em documentos individuais o método split foi utilizado. 
</br>

### JSON após separar as páginas
```
{
  "responses": {
    {
      "context": {
        "pageNumber": 1,
        "uri": "gs://private.linte.io/..."
      }
    },
    {
      "fullTextAnnotation": {
        "text": "text here",
      }
    }
  },
  "fileId": "d8670f2c-443f-4943-9100-647ee0b4c21f"    
}
```

### 3.6 Para tornar os documentos individuais JSON Flat, novos campos foram adicionados(text, uri, pageNumber) e seus respectivos valores foram acessados como valores em objetos. 
</br>

### JSON após separar as páginas
```
{
  "responses": {
    {
      "context": {
        "pageNumber": 1,
        "uri": "gs://private.linte.io/..."
      }
    },
    {
      "fullTextAnnotation": {
        "text": "text here",
      }
    }
  },
  "fileId": "d8670f2c-443f-4943-9100-647ee0b4c21f",
  "text": "text here",
  "pageNumber": 1,
  "uri": "gs://private.linte.io/..."    
}
```

### 3.7 Por fim o campo responses é removido. 
</br>

```
{
  "fileId": "d8670f2c-443f-4943-9100-647ee0b4c21f",
  "text": "text here",
  "pageNumber": 1,
  "uri": "gs://private.linte.io/..."        
}
```

## Indexar PDFs de contrato
</br>

### Na saída do pipeline as credenciais para a conexão com o Elasticsearch são fornecidas.
</br>

```html
output {
  elasticsearch {
     hosts => ["$${ELASTICSEARCH_URL}"]
     index => "hub_search"
     document_id => "%{[@metadata][gcs][name]}"
     routing => "%{fileId}"
     doc_as_upsert => true
     user => "$${ELASTICSEARCH_USERNAME}"
     password => "$${ELASTICSEARCH_PASSWORD}"
  }
}
```
</br>


---
# Reprocessar
## Como definir se um contrato já foi processado pelo logstash?
</br>

### O plugin [logstash-input-google_cloud_storage](https://github.com/logstash-plugins/logstash-input-google_cloud_storage) possui algumas funcionalidades para evitar reconhecer se um arquivo já foi ou não baixado do bucket e enviado para o logstash. Uma delas é incluir um metadado em cada arquivo baixado. O plugin por meio de uma abstração utiliza o gsutil para realizar a adição do metadado com o seguinte comando:
</br>

```
gsutil -m setmeta -h "x-goog-meta-x-goog-meta-ls-gcs-input:processed" gs://private.linte.io/organizations/**/contracts/**/files/**/GoogleVision/*.json
```

### Cada arquivo processado terá um metadado de chave <b>x-goog-meta-ls-gcs-input</b> e valor <b>processed</b>
</br>

## Como reprocessar os contratos?
### O pacote search da aplicação possui um script com as configurações de start do Elasticsearch, localizado em: <b>search/k8s/base/logstash/search.mapping.sh</b>
</br>

```
set -e

waitForUrl() {
    apt update & apt install curl  
    curl --version
    gsutil version
    gcloud auth activate-service-account --key-file "$${GOOGLE_APPLICATION_CREDENTIALS}" # credenciais do gcloud
    gsutil -m setmeta -h "x-goog-meta-x-goog-meta-ls-gcs-input:" "gs://$${GOOGLE_STORAGE_BUCKET_PRIVATE}/organizations/**/contracts/**/files/**/GoogleVision/*.json"
    
    echo "Testing $$1"
    timeout -s TERM 45 sh -c \
    'while [[ "$(curl --user $${ELASTICSEARCH_USERNAME}:$${ELASTICSEARCH_PASSWORD} -s -o /dev/null -L -w ''%{http_code}'' $${0})" != "200" ]];\
    do echo "Waiting for $${0}" && sleep 2;\
    done' $${1}
    
    echo "Running!"

    curl --user $${ELASTICSEARCH_USERNAME}:$${ELASTICSEARCH_PASSWORD} -XDELETE "$${ELASTICSEARCH_URL}/hub_search" -H 'Content-Type: application/json'
    curl --user $${ELASTICSEARCH_USERNAME}:$${ELASTICSEARCH_PASSWORD} -X PUT "$${ELASTICSEARCH_URL}/hub_search" -H 'Content-Type: application/json' -d'
```

### Quando o search é reiniciado, o índice principal hub_search é apagado e em seguida recriado com os mapeamentos adequados para os campos que serão indexados. Sendo assim, quando o search reiniciar precisamos garantir que todos os contratos sejam reprocessados novamente. 
</br>

### Para isso o seguinte comando é utilizado para limpar os metadados dos arquivos:
```
 gsutil -m setmeta -h "x-goog-meta-x-goog-meta-ls-gcs-input:" "gs://$${GOOGLE_STORAGE_BUCKET_PRIVATE}/organizations/**/contracts/**/files/**/GoogleVision/*.json"
```
### O metadado criado pelo plugin "x-goog-meta-ls-gcs-input" tem o valor resetado para nulo. Dessa forma antes do logstash iniciar todos os arquivos do bucket estarão sem os metadados e poderão ser reprocessados.

</br>


---
# Recursos

## A seguir temos as configurações de memória e processamento necessárias para indexar os contratos pelo Logstash:
### <b>search/k8s/base/logstash.deployment.yaml</b>
```
      containers:
        - name: logstash
          resources:
            requests:
              memory: '5Gi'
              cpu: '1000m'
            limits:
              memory: '5Gi'
              cpu: '1000m'
          envFrom:
            - configMapRef:
                name: logstash
            - secretRef:
                name: search-secrets
          env:
            - name: LS_JAVA_OPTS
              value: '-Xms4g -Xmx4g'
```
### <b>Nota: LS_JAVA_OPTS é a configuração responsável por definir o heap size da JVM referente ao logstash. O padrão é 512mb, porém para que o logstash conseguisse carregar os arquivos em memória e pudesse suportar mais eventos por milissegundos o valor passou a ser de 4GB de mínimo e 4gb de máximo. O link abaixo explica o porquê dos números serem iguais para o mínimo e para o máximo.</b>
</br>

### https://www.elastic.co/guide/en/logstash/current/jvm-settings.html#:~:text=The%20recommended%20heap%20size%20for,the%20JVM%20constantly%20garbage%20collecting.
</br>

## Configurações necessárias para indexar os contratos no Elasticsearch
</br>

### <b>search/k8s/base/search.elasticsearch.yaml</b>

```
podTemplate:
  spec:
    containers:
      - name: elasticsearch
        env:
          - name: ES_JAVA_OPTS
            value: "-Xms3g -Xmx3g"      
        resources: 
          requests: 
            memory: "4Gi"      
          limits:   
            memory: "4Gi"
```

### Nota: O ES_JAVA_OPTS é a configuração de heap size do elasticsearch. O padrão é 512mb. Para permitir melhor performance e evitar erro de heap size causado por muitos arquivos carregados em memória, se fez necessário aumentar para 3GB tanto o mínimo quanto o máximo. Mais informações no link abaixo:
</br>

### https://www.elastic.co/guide/en/elasticsearch/reference/current/advanced-configuration.html

</br>

## Como controlar quantos eventos por milissegundos o pipeline de contractFileContent do logstash deve processar?
</br>

### <b>search/k8s/base/logstash/pipelines.yaml</b>

```
- pipeline.id: contract-file-content-pipeline
  path.config: '/usr/share/logstash/pipelines/contractFileContentSync.conf'
  pipeline.batch.size: 30
  pipeline.batch.delay: 60
```
</br>

### O arquivo pipelines.yaml contêm as declarações de todos os pipelines do logstash. É possível definir a nível de pipeline quantos eventos serão processados a cada X milissegundos. No caso de <b>contract-file-content-pipeline</b> serão processados até 30 eventos a cada 60 milissegundos. 
</br>

### <b>Nota: A quantidade de eventos e o tempo devem ser cuidadosamente testados para evitar erros de heap size. Lembre-se 30 conteúdos de arquivos serão carregados em memória a cada 60 milissegundos. <mark>Mais informações no link abaixo:</mark></b>
</br>

### https://www.elastic.co/guide/en/logstash/current/logstash-settings-file.html#logstash-settings-file

</br>

---
# Extras

## Como ver os contratos sendo indexados?
</br>

### Adicionar no pipeline contractContentFile o stdin nas configurações do input:

### <b>/search/k8s/base/logstash/contractContentFileSync.conf</b>

```
input {
  stdin{

  }

  google_cloud_storage {
    interval => 5
    delete => false
    bucket_id => "$${GOOGLE_STORAGE_BUCKET_PRIVATE}"
    json_key_file => "$${GOOGLE_APPLICATION_CREDENTIALS}"   
    file_matches => "organizations\\/.+\\/contracts\\/.+\\/files\\/.+\\/GoogleVision\\/.+\\.json"
    codec => "json"
  }  
}
```
### Adicionar também um stdout no output do pipeline

```
output {
  elasticsearch {
     hosts => ["$${ELASTICSEARCH_URL}"]
     index => "hub_search"
     document_id => "%{[@metadata][gcs][name]}"
     routing => "%{fileId}"
     doc_as_upsert => true
     user => "$${ELASTICSEARCH_USERNAME}"
     password => "$${ELASTICSEARCH_PASSWORD}"
  }
  stdout {
    
  }
}
```

### Na pasta rodar os seguintes comandos para subir a aplicação
```
  cd namespaces/hub/search

  pnpm run deploy:dev
```

### Ao terminar de subir, podemos buscar o nome do pod do logstash utilizando:

```
kubectl get pods -n nome-do-namespace-linte-com-hub
```
### Ao conseguir o nome do pod podemos seguir os logs e observar os contratos sendo indexados, utilizando o comando abaixo:

```
kubectl logs --follow logstash-xxxxxxxxx-xxxxx -n nome-do-namespace-linte-com-hub
```
</br>

## Como instalar plugins no logstash?

### No arquivo logstash.deployment.yaml localizado em <b>search/k8s/base/logstash.deployment.yaml</b> no container do logstash é possível adicionar o plugin no command:

```
containers:
  - name: logstash
    resources:
      requests:
        memory: '5Gi'
        cpu: '1000m'
      limits:
        memory: '5Gi'
        cpu: '1000m'
    envFrom:
      - configMapRef:
          name: logstash
      - secretRef:
          name: search-secrets
    env:
      - name: LS_JAVA_OPTS
        value: '-Xms4g -Xmx4g'
    ports:
      - containerPort: 9600
        name: logstash
    image: docker.elastic.co/logstash/logstash:7.16.3
    volumeMounts:
      - name: secret-volume
        mountPath: /usr/share/logstash/secret-volume
      - name: logstash-lastruns
        mountPath: /usr/share/logstash/lastruns
      - name: logstash-pipelines
        mountPath: /usr/share/logstash/pipelines
      - name: logstash-drivers
        mountPath: /usr/share/logstash/drivers
      - name: logstash-config
        mountPath: /usr/share/logstash/conflogstash.yml
        subPath: logstash.yml
      - name: logstash-config
        mountPath: /usr/share/logstash/confpipelines.yml
        subPath: pipelines.yml
    command:
      [
        'sh',
        '-c',
        '/usr/share/logstash/bin/logstash-pluginstall --no-verify logstash-input-google_cloud_storage logstash-random-plugin-1 logstash-random-plugin-2 && logstash'
      ]
```
### Mais informações sobre como instalar plugins em:
### https://www.elastic.co/guide/en/logstash/current/working-with-plugins.html#installing-plugins

</br>

## Link da prova de conceito de full-text search

### https://linear.app/linte/issue/HUB-428/poc-busca-avancada


