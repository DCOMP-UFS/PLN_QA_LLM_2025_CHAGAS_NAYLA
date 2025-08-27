## Guia sobre a avaliação de modelos de linguagem para tarefas de Pergunta e Resposta (QA) em português brasileiro.

**Equipe:**

    Camille: seleção de modelos; implementação; validação da documentação.
    Nayla: seleção de modelos; seleção de perguntas; implementação; documentação. 
    Túlio: seleção de modelos; implementação; testes; gravação do vídeo.

Neste tutorial, vamos explorar um processo estruturado para testar e comparar o desempenho de modelos de código aberto, disponíveis gratuitamente na plataforma Hugging Face. Nosso principal objetivo é responder à pergunta: **"Como é o desempenho de modelos gratuitos do Hugging Face na tarefa de QA em português brasileiro?"**

Para isso, utilizaremos um notebook em Python, criado no ambiente do Google Colab, e dois documentos que servirão como nossa base de conhecimento: um sobre doenças respiratórias crônicas e um dicionário de dados do sus. 

### Processo de avaliação

#### **Etapa 1: Configuração do ambiente de trabalho**

O primeiro passo em qualquer projeto de processamento de linguagem natural é garantir que todas as ferramentas necessárias estejam instaladas. No ambiente do Google Colab, isso é feito de maneira simples, utilizando o comando:

```python
!pip install -U --quiet transformers pypdf python-docx pandas
```

  * **`transformers`**: Esta é a biblioteca principal do Hugging Face. Ela nos dá acesso a milhares de modelos pré-treinados para diversas tarefas, incluindo a de Perguntas e Respostas.
  * **`pypdf`**: Utilizada para extrair o texto de documentos no formato PDF. No nosso caso, será usada para ler o arquivo `doencas_respiratorias_cronicas.pdf`.
  * **`python-docx`**: Permite a extração de texto de documentos do Microsoft Word (`.docx`), como o nosso `DICIONARIO_DE_DADOS.docx`.
  * **`pandas`**: Embora não seja central para a tarefa de QA em si, é uma ferramenta poderosa para manipulação e análise de dados, útil para organizar os resultados dos testes.

Após a instalação, importamos os módulos necessários para o nosso script.

```python
import pypdf
import docx
import pandas as pd
from transformers import pipeline
```

#### **Etapa 2: Construção da base de conhecimento**

Um modelo de QA precisa de um contexto para encontrar as respostas. Nesta etapa, preparamos nossos dois documentos para que sirvam como essa base de conhecimento, extraindo o texto bruto e armazenando-o em variáveis.

**`doencas_respiratorias_cronicas.pdf`**

Utiliza-se a biblioteca `pypdf` para ler o arquivo PDF página por página e concatenar todo o texto em uma única variável chamada `conteudo_doencas_cronicas`.

```python
doencas_cronicas = pypdf.PdfReader("doencas_respiratorias_cronicas.pdf")
conteudo_doencas_cronicas = "\\n".join([doencas_cronicas.pages[i].extract_text() for i in range(len(doencas_cronicas.pages))])
```

**`DICIONARIO_DE_DADOS.docx`**

De forma similar à pypdf, a biblioteca `docx` é percorre o documento do Word, extraindo o conteúdo de parágrafos e tabelas. O texto de cada célula da tabela é unido por um separador (`|`) para manter a estrutura, e o resultado final é armazenado na variável `dict_dados`.

```python
doc = docx.Document("DICIONARIO_DE_DADOS.docx")
textos_completos = []
for para in doc.paragraphs:
    textos_completos.append(para.text)
for table in doc.tables:
    for row in table.rows:
        textos_completos.append(" | ".join(cell.text for cell in row.cells))
dict_dados = "\\n".join(textos_completos)
```

-----

#### **Etapa 3: Modularização da função de teste**

Para organizar o processo de avaliação, foi criada uma função reutilizável chamada `responder_pergunta`. Esta função é o coração do nosso experimento. Ela recebe dois argumentos: qa_input e model_name.

```python
def responder_pergunta(qa_input: dict, model_name: str) -> dict:
    """
    Carrega um modelo de Question Answering e retorna a resposta para uma pergunta.
    """
    qa_model = pipeline("question-answering", model=model_name, tokenizer=model_name)
    resultado = qa_model(qa_input)

    return resultado
```

Os argumentos da função possuem os seguintes papéis:
      * `qa_input`: Um dicionário contendo a pergunta (`question`) e o texto de referência (`context`).
      * `model_name`: O nome do modelo a ser baixado do Hugging Face.

Dentro dela, inicializamos o `pipeline`, executamos o modelo com a pergunta desejada e o contexto e retornamos a resposta gerada.

>  O `pipeline` do Hugging Face é uma abstração de alto nível que simplifica o uso de modelos. Ao especificar `"question-answering"`, a biblioteca cuida de todo o pré-processamento do texto e da formatação da saída.

 A saída da função é um dicionário que contém a resposta encontrada (`answer`), um índice de confiança (`score`) e a localização da resposta no texto original (`start` e `end`).

#### **Etapa 4: Avaliação do desempenho**

Com o ambiente e as funções prontos, o processo de avaliação pode começar. Nosso objetivo é submeter três modelos diferentes às mesmas perguntas e comparar suas respostas.

**1. Seleção dos modelos:**

Para este teste, foram escolhidos três modelos gratuitos disponíveis no Hugging Face. A seleção levou em conta a popularidade e a adequação para a tarefa, mas também as limitações do ambiente (Google Colab gratuito), que restringe o uso de modelos muito grandes. Os modelos selecionados para o teste foram:

  * `deepset/xlm-roberta-large-squad2`
  * `pierreguillou/bert-base-cased-squad-v1.1-portuguese`
  * `benjleite/ptt5-ptbr-qa`

**2. Escolha das perguntas:**

Foram selecionadas manualmente três perguntas para cada documento, buscando cobrir diferentes tipos de extração de informação.

  * **Para o documento `doencas_respiratorias_cronicas.pdf`:**
    1.  O que diferencia a hemoptise falsa da verdadeira?
    2.  Quais os tratamentos e atos de prevenção para rinite persistente grave?
    3.  Quais os cuidados ambientais necessários para o controle das crises asmáticas?
  * **Para o documento `DICIONARIO_DE_DADOS.docx`:**
    1.  Quais são os possíveis tipos de serviço referenciado, de acordo com a tabela `LFCES019`?
    2.  Que tabelas se relacionam com a `tb_carga_horaria_sus`?
    3.  Recebi um arquivo de planilha via email, que alegava ser uma amostra da tabela `rl_estab_serv_class`. Ao investigar o arquivo, notei que a planilha apresentava a coluna `codservico` com alguns registros nulos. Posso considerar esse arquivo como uma amostra verídica?

**3. Execução dos Testes:**

Para cada uma das seis perguntas, o processo é o mesmo:

  * Monta-se o dicionário `qa_input` com a pergunta e o contexto apropriado.
  * Executa-se a função `responder_pergunta` três vezes, uma para cada `model_name` selecionado.
  * As respostas de cada modelo são armazenadas para análise posterior.

### Resultados apresentados

Reunimos as respostas apresentadas por cada modelo, bem como as respostas corretas para cada uma das perguntas selecionadas, e construímos uma tabela de comparação. Os resultados do experimento podem ser vistos abaixo.

| Pergunta | Documento base | deepset/xlm-roberta-large-squad2 | pierreguillou/bert-base-cased-squad-v1.1-portuguese | benjleite/ptt5-ptbr-qa | Resposta esperada |
| --------------- | -------- | -------- | -------- | -------- | ----------------- |
| O que diferencia a hemoptise falsa da verdadeira? | doencas_respiratorias_cronicas.pdf | ``' o sangramento'`` | ``'Níveis de evidência'`` | ``' de'`` | Na verdadeira hemoptise, a origem do sangue está nos vasos da parede da traqueia, brônquios ou do tecido pulmonar, enquanto na falsa, o sangramento se localiza nas vias aéreas superiores ou no trato digestivo superior. Diferentemente da falsa, na verdadeira hemoptise o sangue habitualmente tem aspecto vivo e rutilante, é espumoso e está misturado a alguma quantidade de muco. |
| Quais os tratamentos e atos de prevenção para rinite persistente grave? | doencas_respiratorias_cronicas.pdf | ``' SAMU \n192'`` | ``'Corticosteroids'`` | ``' de'`` | Corticoide tópico nasal por pelo menos 60 dias. Abordagem educacional, uso correto das medicações, cessação do tabagismo, prevenção do sobrepeso, atividades físicas e controle ambiental. |
| Quais os cuidados ambientais necessários para o controle das crises asmáticas? | doencas_respiratorias_cronicas.pdf | ``'\nAtividades educativas + controle ambiental'`` | ``'Atividades educativas'`` | ``' (>'`` | Não há evidências científicas robustas para embasar recomendações generalizadas para controle do ambiente domiciliar no paciente com asma. No entanto, intervenções múltiplas conjuntas para limpeza domiciliar e métodos físicos para o controle da exposição ao ácaro têm demonstrado algum benefício. |
| Quais são os possíveis tipos de serviço referenciado, de acordo com a tabela `LFCES019`? | DICIONARIO_DE_DADOS.docx | ``'\nTABELA DE SERVIÇO REFERENCIADO'`` | ``'Diálise\n2 - Quimioterapia e Radioterapia'`` | ``' Uso'`` | Diálise (1), Quimioterapia e Radioterapia (2), Hemoterapia (3)|
| Que tabelas se relacionam com a `tb_carga_horaria_sus`? | DICIONARIO_DE_DADOS.docx | ``' VÍNCULOS DO PROFISSIONAL NO ESTABELECIMENTO'`` | ``'NOME DO CAMPO'`` | ``' DT_CMTP_FIM / DATE /  /  / NULL / Data Final'`` | TB_ESTABELECIMENTO, TB_ATIVIDADE_PROFISSIONAL, TB_MOD_VINCULO, TB_TP_MOD_VINCULO, TB_SUB_TP_MOD_VINCULO |
| Recebi um arquivo de planilha via email, que alegava ser uma amostra da tabela `rl_estab_serv_class`. Ao investigar o arquivo, notei que a planilha apresentava a coluna `codservico` com alguns registros nulos. Posso considerar esse arquivo como uma amostra verídica? | DICIONARIO_DE_DADOS.docx | ``' VARCHAR(7)'`` | ``'NULL / Data da Primeira entrada no Banco de Produção Federal'`` | ``' TIPO / FOREIGN'`` | Não. O campo codservico não pode ser nulo. |

O modelo que se mostrou mais promissor nas perguntas de ambos os documentos foi o `deepset/xlm-roberta-large-squad2`, que das 6 perguntas, trouxe uma resposta relativamente próxima ao esperado em 3 delas. Seguido pelo `pierreguillou/bert-base-cased-squad-v1.1-portuguese`, que trouxe 2 e por último, o modelo `benjleite/ptt5-ptbr-qa` que não apresentou nenhuma resposta satisfatória.

Os resultados apresentados são diretamente influenciados pela estrutura das perguntas. Como são perguntas mais contextuais, a performance de modelos recuperadores (retriever) em selecionar uma resposta explicitamente exata se torna muito pobre. Além disso, a quantidade de parâmetros dos modelos escolhidos também impede a recuperação de respostas densas.

### Proposta de utilização

Para exemplificar o ponto levantado acima, implementamos um RAG utilizando os mesmos documentos como base de conhecimento, mas usando um modelo de maior número de parâmetros e de estrutura geradora.

#### Etapas de implementação

##### Instalação de dependências adicionais

Novamente, iniciamos o processo instalando as bibliotecas necessárias.

```python
!pip install --quiet transformers torch sentencepiece faiss-cpu pypdf python-docx langchain sentence-transformers
```
Os pacotes importados possuem os seguintes objetivos:

* **sentence-transformers**: gera representações vetoriais (embeddings) de textos.
* **faiss-cpu**: provê índice vetorial eficiente para busca por similaridade.
* **langchain**: utiliza-se apenas o *text splitter* para fracionar documentos.
* **torch** e **transformers**: continuam sendo a base para carregamento/execução de modelos.

Essas adições não substituem o fluxo anterior; elas o complementam para que as respostas passem a considerar trechos relevantes do corpus antes da geração.

##### Importações e organização do ambiente

Em seguida, são importados os módulos utilizados:

```python
import torch
import faiss
import numpy as np
from transformers import AutoTokenizer, AutoModel, AutoModelForSeq2SeqLM, AutoModelForCausalLM
from sentence_transformers import SentenceTransformer
from langchain.text_splitter import RecursiveCharacterTextSplitter
```
Os módulos possuem os seguintes propósitos:

* **SentenceTransformer**: cria embeddings semânticos multilíngues.
* **FAISS**: cria e consulta o índice vetorial.
* **Transformers (AutoTokenizer / AutoModel\*)**: carrega o modelo gerador (seq2seq ou causal) que produzirá a resposta final a partir do contexto recuperado.
* **Torch**: define o *device* (CPU/GPU) para acelerar a inferência.

##### Fragmentação dos documentos (*chunking*)

A base de conhecimento já construída (variáveis `conteudo_doencas_cronicas` e `dict_dados`) é fracionada em blocos com sobreposição:

```python
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=100,
    length_function=len,
    separators=["\n\n", "\n", ". ", " ", ""]
)

chunks_doencas = text_splitter.split_text(conteudo_doencas_cronicas)
chunks_dict    = text_splitter.split_text(dict_dados)
```

* **`chunk_size=500`**: tamanho alvo de cada trecho.
* **`chunk_overlap=100`**: sobreposição entre blocos, reduzindo cortes semânticos.
* **Separadores**: preservam limites naturais (parágrafos, sentenças) antes de cortar por tamanho.

Esse processo permite que os mesmos textos que extraímos na etapa anterior do experimento possam ser utilizados de forma contextual, a partir da verificação de similaridade.

##### Modelo de embeddings e seleção de dispositivo

Define-se o *device* e o modelo de embeddings multilíngue:

```python
device   = "cuda" if torch.cuda.is_available() else "cpu"
embedder = SentenceTransformer("intfloat/multilingual-e5-large", device=device)
```

O modelo escolhido é adequado a português e fornece vetores semânticos robustos para busca densa.

##### Função utilitária para gerar embeddings

Cria-se uma função (ex.: `get_embeddings`) que recebe uma lista de textos e devolve uma matriz `numpy` de embeddings. Internamente, utiliza `model.encode(...)` com batching para eficiência. Essa função padroniza a vetorização tanto dos chunks quanto das perguntas, garantindo que tudo esteja no mesmo espaço semântico.


##### Indexação vetorial com FAISS

A partir dos embeddings dos `chunks_*`, constroem-se índices FAISS separados para cada documento (nomenclatura típica no notebook):

* `index_doencas` para `chunks_doencas`
* `index_dict` para `chunks_dict`

O procedimento geral envolve:

1. Geração dos embeddings dos *chunks* com `get_embeddings(...)`.
2. Criação do índice FAISS (por exemplo, um índice denso como `IndexFlatL2`).
3. Inserção dos vetores dos *chunks* no índice correspondente.

A criação de dois índices mantêm a separação lógica dos corpora, refletindo a organização original (documento clínico vs. dicionário de dados). Isso permite comparar o RAG com as perguntas já categorizadas por fonte.

##### Carregamento do modelo gerador

Carrega-se um tokenizer e um modelo gerador (seq2seq ou causal) via `AutoTokenizer.from_pretrained(...)` e `AutoModelForSeq2SeqLM` ou `AutoModelForCausalLM`. A variável resultante (por exemplo, `generator`) é usada para produzir a resposta textual condicionada ao prompt com o contexto recuperado.

##### Função de retrieval + geração: `qa_rag(...)`

Define-se a função principal, responsável por:

1. **Vetorização da pergunta**: `embedding_pergunta = get_embeddings([pergunta], embedder)`.
2. **Busca no índice**: `_, I = index.search(embedding_pergunta, k)`, onde `k` é o número de trechos retornados (padrão `k=5`).
3. **Composição do contexto**: seleção de `chunks_list[i]` para `i` em `I[0]` e junção em um único texto.
4. **Construção do *prompt***: instruções claras em português para usar apenas o contexto; caso não haja evidências, retornar “Não encontrei no contexto.”.
5. **Geração**: chamada ao `generator(...)` com hiperparâmetros conservadores:

   * `max_length=1024`
   * `temperature=0.2`
   * `top_p=0.85`
   * `num_return_sequences=1`
6. **Pós-processamento**: extração do trecho após o marcador **"Resposta:"** no texto gerado.

Trecho do prompt utilizado na função:

```
Você é um especialista.
Use APENAS as informações do contexto para responder em português claro, objetivo e resumido.
Se não encontrar no contexto, diga: "Não encontrei no contexto."

Contexto:
{...}

Pergunta: {pergunta}

Resposta:
```

Diferentemente dos modelos extrativos que testamos antes, a resposta aqui é gerada pelo modelo, porém ancorada nos trechos selecionados por similaridade, o que atende ao objetivo de avaliar se um esquema RAG melhora a precisão e a fidelidade às fontes.

##### Execução com o mesmo conjunto de perguntas

Por fim, reusa-se o mesmo conjunto de perguntas definidas anteriormente, iterando sobre as listas:

```python
for pergunta in qa_doencas_respiratorias:
    resposta = qa_rag(pergunta, index_doencas, chunks_doencas)
    print(f"Pergunta: {pergunta}\nResposta: {resposta}\n\n")

for pergunta in qa_dic_dados:
    resposta = qa_rag(pergunta, index_dict, chunks_dict)
    print(f"Pergunta: {pergunta}\nResposta: {resposta}\n\n")
```

* **`qa_doencas_respiratorias`**: perguntas vinculadas ao PDF clínico.
* **`qa_dic_dados`**: perguntas vinculadas ao dicionário de dados.

##### Observações práticas para análise dos resultados

* **Aderência ao contexto**: verificar se a resposta menciona elementos presentes nos *chunks* recuperados.
* **Sinalização de ausência**: confirmar o uso de *“Não encontrei no contexto.”* quando apropriado.
* **Sensibilidade a `k` e ao *chunking***: variações em `k`, `chunk_size` e `chunk_overlap` podem alterar a cobertura do contexto; recomenda-se documentar valores usados e rationale.
* **Limitações do gerador**: modelos menores podem sintetizar respostas mais curtas; manter `max_length` e *temperatura* baixos tende a favorecer objetividade.


#### Resultados obtidos

| Pergunta | Documento base | Modelo QA | Modelo com RAG | Resposta esperada |
| --------------- | -------- | -------- | -------- | ----------------- |
| O que diferencia a hemoptise falsa da verdadeira? | doencas_respiratorias_cronicas.pdf |``' o sangramento'`` |  A hemoptise falsa tem origem nas vias aéreas superiores ou no trato digestivo superior, enquanto a verdadeira tem origem nos vasos da parede da traqueia, brônquios ou do tecido pulmonar. Além disso, o sangue na hemoptise verdadeira é mais espumoso, vivo e rutilante, misturado a muco, enquanto na falsa há sangramento nas vias aéreas superiores ou no trato digestivo superior, com coloração mais escura e sem mistura com muco. Também, a hemoptise verdadeira apresenta escarro purulento ou mucopurulento e amarelado ou esverdeado, frequentemente associada a quadros infecciosos. Já a hemoptise falsa não apresenta esses sinais. |  Na verdadeira hemoptise, a origem do sangue está nos vasos da parede da traqueia, brônquios ou do tecido pulmonar, enquanto na falsa, o sangramento se localiza nas vias aéreas superiores ou no trato digestivo superior. Diferentemente da falsa, na verdadeira hemoptise o sangue habitualmente tem aspecto vivo e rutilante, é espumoso e está misturado a alguma quantidade de muco. |
| Quais os tratamentos e atos de prevenção para rinite persistente grave? | doencas_respiratorias_cronicas.pdf | ``' SAMU \n192'`` | 1. Medicamentos: Corticoide inalatório nasal, Anti-histamínico H1 oral (nas doses especificadas). Atos de Prevenção: Reduzir a exposição a fatores desencadeantes individuais, Evitar exposição a ácaros ou alérgenos, Evitar exposição a mofo, Evitar tabagismo ativo e passivo, Retirar animais domésticos se comprovada sensibilização, Evitar odores fortes e exposição ocupacional, Evitar locais de poluição atmosférica. Não encontrei no contexto. Não há informações específicas sobre atos de prevenção direcionados para rinite persistente grave no texto fornecido. O tratamento específico para essa condição é focado em medicamentos inalatórios e orais. Para rinite persistente grave, o tratamento deve ser prolongado por pelo menos 60 dias e reavaliado após uma semana. | Corticoide tópico nasal por pelo menos 60 dias. Abordagem educacional, uso correto das medicações, cessação do tabagismo, prevenção do sobrepeso, atividades físicas e controle ambiental. |
| Quais os cuidados ambientais necessários para o controle das crises asmáticas? | doencas_respiratorias_cronicas.pdf | ``'\nAtividades educativas + controle ambiental'`` |  Não encontrei no contexto cuidados ambientais específicos para o controle das crises asmáticas. O texto menciona que há poucas evidências científicas para recomendações gerais sobre controle do ambiente domiciliar no paciente com asma, destacando que nenhuma medida simples é eficaz em reduzir a exposição a alérgenos do ácaro. |  Não há evidências científicas robustas para embasar recomendações generalizadas para controle do ambiente domiciliar no paciente com asma. No entanto, intervenções múltiplas conjuntas para limpeza domiciliar e métodos físicos para o controle da exposição ao ácaro têm demonstrado algum benefício. |
| Quais são os possíveis tipos de serviço referenciado, de acordo com a tabela `LFCES019`? | DICIONARIO_DE_DADOS.docx |  ``'\nTABELA DE SERVIÇO REFERENCIADO'`` | _MUN_ADM_GERENCIA_CNES / GERENTES/ADMINISTRADORES X CONTRATOS DA UNIDADE MUNICIPAL / LFCES096 / RL_ESTAB_CONTRATO / CONTRATOS DO ESTABELECIMENTO / LFCES097 / RL_MUN_CONTRATO / CONTRATOS DA UNIDADE MUNICIPAL / LFCES098 / RL_ESTAB_CONTRATO_ULT / CONTRATOS DO ESTABELECIMENTO ÚLTIMO PERÍODO / LFCES099 / RL_MUN_CONTRATO_ULT / CONTRATOS DA UNIDADE MUNICIPAL ÚLTIMO PERÍODO / LFCES100 / RL_ESTAB_CONTRATO_ANT / CONTRATOS DO ESTABELECIMENTO ANTERIOR / LFCES101 / RL_MUN_CONTRATO_ANT / CONTRATOS DA UNIDADE MUNICIPAL ANTERIOR / LFCES102 / RL_ESTAB_CONTRATO_ANT_ULT / CONTRATOS DO ESTABELECIMENTO ANTERIOR ÚLTIMO PERÍODO / LFCES103 | Diálise (1), Quimioterapia e Radioterapia (2), Hemoterapia (3) |
| Que tabelas se relacionam com a `tb_carga_horaria_sus`? | DICIONARIO_DE_DADOS.docx | ``' VÍNCULOS DO PROFISSIONAL NO ESTABELECIMENTO'`` | Não há outras tabelas mencionadas que se relacionem diretamente com TB_CARGA_HORARIA_SUS no contexto fornecido. Não encontrei no contexto. Toquei nas três tabelas mencionadas, mas não identifiquei nenhuma outra relacionada especificamente à TB_CARGA_HORARIA_SUS. | TB_ESTABELECIMENTO, TB_ATIVIDADE_PROFISSIONAL, TB_MOD_VINCULO, TB_TP_MOD_VINCULO, TB_SUB_TP_MOD_VINCULO |
| Recebi um arquivo de planilha via email, que alegava ser uma amostra da tabela `rl_estab_serv_class`. Ao investigar o arquivo, notei que a planilha apresentava a coluna `codservico` com alguns registros nulos. Posso considerar esse arquivo como uma amostra verídica? | DICIONARIO_DE_DADOS.docx | ``' VARCHAR(7)'`` | Para determinar se o arquivo pode ser considerado uma amostra verídica da tabela `rl_estab_serv_class`, seria necessário verificar se os campos presentes no arquivo estão corretamente preenchidos e se os valores nulos correspondem aos esperados para essa tabela. No contexto fornecido, não há informações específicas sobre a tabela `rl_estab_serv_class` ou quaisquer campos nela relacionados à coluna `codservico`. Portanto, não podemos fazer uma conclusão baseada apenas nas informações fornecidas. Seria necessário analisar diretamente a estrutura da tabela `rl_estab_serv_class` e verificar os valores nulos em `codservico` para determinar se eles são aceitáveis ou não. Além disso, seria importante verificar se os outros campos do arquivo estão consistentes com a estrutura da tabela `rl_estab_serv_class`.  | Não. O campo codservico não pode ser nulo. |

#### Comparação de Desempenho dos Modelos Testados

| Modelo | Total de Perguntas |  Acertos Sobre Doenças Respiratórias | Acertos Sobre Dicionário de Dados | Taxa de Sucesso |
| --------------- | -------- | -------- | -------- | ----------------- |
| deepset/xlm-roberta-large-squad2 | 6 | 2 | 1 | 50% |
| pierreguillou/bert-base-cased-squad-v1.1-portuguese | 6 | 1 | 1 | 33% |
| benjleite/ptt5-ptbr-qa | 6  | 0 | 0 | 0% |
| Qwen/Qwen2.5-3B-Instruct | 6 | 3 | 0 | 50% |

Observando os resultados acima, podemos perceber que o modelo com RAG entrega respostas mais completas em relação ao documento de doenças respiratórias crônicas. Mas ainda se mostra ineficiente ao buscar respostas no dicionário de dados.
