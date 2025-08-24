## Guia sobre a avaliação de modelos de linguagem para tarefas de Pergunta e Resposta (QA) em português brasileiro.

**Equipe:**

    CAMILLE SOUSA MENESES DE SANTANA
    NAYLA SAHRA SANTOS DAS CHAGAS
    TULIO SOUSA DE GOIS

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



### Proposta de utilização

Para aprimorar ainda mais a performance do modelo `deepset/xlm-roberta-large-squad2` na função de QA, implementamos o modelo numa abordagem RAG.

#### Etapas de implementação

#### Resultados obtidos

| Pergunta | Documento base | Modelo QA | Modelo com RAG | Resposta esperada |
| --------------- | -------- | -------- | -------- | ----------------- |
| O que diferencia a hemoptise falsa da verdadeira? | doencas_respiratorias_cronicas.pdf | Modelo 1 | Modelo 2 |  Na verdadeira hemoptise, a origem do sangue está nos vasos da parede da traqueia, brônquios ou do tecido pulmonar, enquanto na falsa, o sangramento se localiza nas vias aéreas superiores ou no trato digestivo superior. Diferentemente da falsa, na verdadeira hemoptise o sangue habitualmente tem aspecto vivo e rutilante, é espumoso e está misturado a alguma quantidade de muco. |
| Quais os tratamentos e atos de prevenção para rinite persistente grave? | doencas_respiratorias_cronicas.pdf | Modelo 1 | Modelo 2 | Corticoide tópico nasal por pelo menos 60 dias. Abordagem educacional, uso correto das medicações, cessação do tabagismo, prevenção do sobrepeso, atividades físicas e controle ambiental. |
| Quais os cuidados ambientais necessários para o controle das crises asmáticas? | doencas_respiratorias_cronicas.pdf | Modelo 1 | Modelo 2 |  Não há evidências científicas robustas para embasar recomendações generalizadas para controle do ambiente domiciliar no paciente com asma. No entanto, intervenções múltiplas conjuntas para limpeza domiciliar e métodos físicos para o controle da exposição ao ácaro têm demonstrado algum benefício. |
| Quais são os possíveis tipos de serviço referenciado, de acordo com a tabela `LFCES019`? | DICIONARIO_DE_DADOS.docx | Modelo 1 | Modelo 2 | Diálise (1), Quimioterapia e Radioterapia (2), Hemoterapia (3) |
| Que tabelas se relacionam com a `tb_carga_horaria_sus`? | DICIONARIO_DE_DADOS.docx | Modelo 1 | Modelo 2 | TB_ESTABELECIMENTO, TB_ATIVIDADE_PROFISSIONAL, TB_MOD_VINCULO, TB_TP_MOD_VINCULO, TB_SUB_TP_MOD_VINCULO |
| Recebi um arquivo de planilha via email, que alegava ser uma amostra da tabela `rl_estab_serv_class`. Ao investigar o arquivo, notei que a planilha apresentava a coluna `codservico` com alguns registros nulos. Posso considerar esse arquivo como uma amostra verídica? | DICIONARIO_DE_DADOS.docx | Modelo 1 | Modelo 2 | Não. O campo codservico não pode ser nulo. |