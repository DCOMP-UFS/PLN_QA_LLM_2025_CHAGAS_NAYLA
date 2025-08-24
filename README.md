## Guia sobre a avaliação de modelos de linguagem para tarefas de Pergunta e Resposta (QA) em português brasileiro.

**Equipe:**

    CAMILLE SOUSA MENESES DE SANTANA
    NAYLA SAHRA SANTOS DAS CHAGAS
    TULIO SOUSA DE GOIS

Neste tutorial, vamos explorar um processo estruturado para testar e comparar o desempenho de modelos de código aberto, disponíveis gratuitamente na plataforma Hugging Face. Nosso principal objetivo é responder à pergunta: **"Como é o desempenho de modelos gratuitos do Hugging Face na tarefa de QA em português brasileiro?"**

Para isso, utilizaremos um notebook em Python, criado no ambiente do Google Colab, e dois documentos que servirão como nossa base de conhecimento: um sobre doenças respiratórias crônicas e um dicionário de dados do sus. 

### Processo de avaliação

#### **Etapa 1: Configuração do ambiente de trabalho**

O primeiro passo em qualquer projeto de processamento de linguagem natural é garantir que todas as ferramentas necessárias estejam instaladas.

#### **Etapa 2: Construção da base de conhecimento**

Um modelo de QA precisa de um contexto para encontrar as respostas. Nesta etapa, preparamos nossos dois documentos para que sirvam como essa base de conhecimento, extraindo o texto bruto e armazenando-o em variáveis.

#### **Etapa 3: Avaliação do desempenho**

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
