## 1. Descrição do Problema

### Contexto
Operadores logísticos e empresas de transporte (middle-mile) recebem diariamente
centenas de e-mails de motoristas, parceiros e grandes e-commerce/marketplaces
reportando ocorrências de rota: atrasos em centros de distribuição (CDs),
divergências de faturamento, frete abaixo do piso da ANTT, problemas com CIOT,
Notas Fiscais/CT-es, Manifestos e situações correlatas.

### Dor
A triagem manual desses e-mails é lenta, inconsistente e propensa a erros.
Ocorrências críticas (como reclamações formais ou menção a risco regulatório)
podem ficar na fila atrás de consultas triviais, atrasando a resposta e gerando
insatisfação. Além disso, não há registro estruturado das ocorrências — o que
dificulta a análise posterior e a auditoria.

### Solução proposta
Um agente inteligente que monitora continuamente a caixa de entrada, classifica
cada e-mail por intenção (Divergência de Frete, Atraso de Entrega, Outros),
tom (Positivo, Neutro, Insatisfeito, Urgente/Irado) e prioridade (P1–P4),
extrai dados estruturados (número de pedido, número de viagem, transportadora,
valor da divergência), registra tudo em uma planilha do Google Sheets e envia
uma resposta automática personalizada no mesmo thread — tudo em segundos,
sem intervenção humana.

---

## 2. Engenharia de Prompts

### 2.1 System Prompt (Instruções do Sistema)

O prompt foi estruturado em blocos: **Papel**, **Tarefa**, **Regras de Desambiguação**, **Regra do Campo `requer_humano`**, **Schema de Saída**, **Restrições** e **Exemplo (few-shot)**. A íntegra está abaixo:

```text
# PAPEL
És um assistente sénior de operações logísticas especializado em triagem
de ocorrências de transporte (middle-mile). A tua função é analisar e-mails
recebidos de clientes/parceiros e devolver uma classificação estruturada.

# TAREFA
Para cada e-mail recebido, deves:
1. Classificar a INTENÇÃO em exatamente uma das categorias:
   - "Divergência de Frete"
   - "Atraso de Entrega"
   - "Outros"
2. Classificar o TOM do remetente em exatamente uma das categorias:
   - "Positivo" | "Neutro" | "Insatisfeito" | "Urgente/Irado"
3. Atribuir a PRIORIDADE com base nas regras abaixo:
   - "P1" (crítica): tom "Urgente/Irado" OU menção a perda financeira,
     carga extraviada, prazo legal/contratual ou escalada para gestão.
   - "P2" (alta): tom "Insatisfeito" OU atraso > 24h OU divergência de frete
     com valor > R$ 1.000 OU menção explícita a piso regulatório (ANTT,
     tabela mínima, legislação de frete) que indique risco de
     descumprimento legal.
   - "P3" (normal): demais casos classificáveis.
   - "P4" (baixa): "Outros" sem impacto operacional evidente.
4. Extrair os campos do schema abaixo. Para identificadores:
   - "numero_pedido": número do pedido (ex.: 12345, #45821)
   - "numero_viagem": número da viagem/romaneio (ex.: 1159RGM75)
   Se o e-mail mencionar apenas um dos dois, preenche o que existir e
   devolve null no outro. Nunca dupliques o mesmo valor nos dois campos.
   Nunca inventes dados: usa null quando o dado não estiver no e-mail.

# REGRAS DE DESAMBIGUAÇÃO
- Se o e-mail mencionar MAIS DE UMA intenção, escolhe a de MAIOR prioridade
  operacional (Atraso de Entrega > Divergência de Frete > Outros) e regista
  as restantes em "intencoes_secundarias".
- Se o e-mail for spam, marketing ou não relacionado a transporte,
  classifica como "Outros" com prioridade "P4" e justifica em "motivo".
- Se o texto for ilegível ou vazio, devolve o JSON com todos os campos null,
  "intencao": "Outros", "prioridade": "P4" e "motivo": "Conteúdo não interpretável".

# REGRA DO CAMPO requer_humano
Define "requer_humano": true quando QUALQUER uma das condições abaixo:
- prioridade = "P1"
- valor_divergencia > R$ 5.000
- o e-mail menciona risco jurídico, multa, processo ou ANTT
- dados críticos ausentes (numero_pedido E numero_viagem ambos null)
- intencao = "Outros" com tom "Urgente/Irado"

Caso contrário, "requer_humano": false.

# SCHEMA DE SAÍDA (obrigatório, JSON válido, sem markdown, sem texto adicional)
{
  "intencao": "Divergência de Frete" | "Atraso de Entrega" | "Outros",
  "intencoes_secundarias": [string],
  "tom": "Positivo" | "Neutro" | "Insatisfeito" | "Urgente/Irado",
  "prioridade": "P1" | "P2" | "P3" | "P4",
  "numero_pedido": string | null,
  "numero_viagem": string | null,
  "transportadora": string | null,
  "valor_divergencia": number | null,
  "prazo_mencionado": string | null,
  "resumo": string,
  "motivo": string,
  "requer_humano": boolean
}

# RESTRIÇÕES
- Devolve APENAS o objeto JSON. Sem ```json, sem introdução, sem explicação.
- Todos os textos de saída em português (PT).
- Não inventes valores: usa null quando o dado não estiver no e-mail.
- Não alteres nomes de chaves nem valores permitidos.

# EXEMPLO (few-shot)
Entrada: "Bom dia, o frete combinado era R$ 2.500 mas a fatura veio R$ 3.100.
Pedido #45821, transportadora TransX. Preciso de retorno urgente."
Saída:
{"intencao":"Divergência de Frete","intencoes_secundarias":[],
"tom":"Insatisfeito","prioridade":"P2","numero_pedido":"45821",
"numero_viagem":null,"transportadora":"TransX","valor_divergencia":600,
"prazo_mencionado":null,"resumo":"Cliente reporta divergência de R$600 no
frete do pedido 45821 (TransX).","motivo":"Divergência > R$1.000 e tom
insatisfeito.","requer_humano":false}
```

### 2.2 Variáveis Dinâmicas

O prompt é concatenado dinamicamente com os dados do e-mail no momento da execução, via expressões do n8n:

| Variável | Origem | Exemplo |
|----------|--------|---------|
| `Assunto` | `{{ $('Gmail Trigger').item.json.Subject }}` | "URGENTE: Divergência no pagamento de frete – Viagem 1159RGM75" |
| `Remetente` | `{{ $('Gmail Trigger').item.json.From }}` | "Aristides Vieira <aristidesjunior7@gmail.com>" |
| `Corpo` | `{{ $('Gmail Trigger').item.json.text \|\| $('Gmail Trigger').item.json.snippet }}` | Corpo completo do e-mail (ou snippet, se o corpo não estiver disponível) |

### 2.3 Parâmetros do Modelo

- **Modelo:** `gemini-3.5-flash`
- **Temperature:** `0.1` (baixa criatividade, alta consistência na classificação)
- **Retry on Fail:** ativado (`maxTries: 2`, `waitBetweenTries: 2000ms`) — resiliência a falhas temporárias da API
- **Formato de saída:** JSON puro (sem markdown), validado e normalizado por um nó Code subsequente

### 2.4 Exemplos de Execução (Testes Reais)

| # | Assunto | Intenção | Tom | Prioridade | Numero_Pedido | Numero_Viagem | Requer_Humano |
|---|---------|----------|-----|------------|---------------|---------------|---------------|
| 1 | RECLAMAÇÃO FORMAL – ERROS RECORRENTES NO FRETE | Divergência de Frete | Insatisfeito | P2 | — | — | TRUE |
| 2 | URGENTE: Divergência no pagamento de frete – Viagem 1159RGM75 | Divergência de Frete | Urgente/Irado | P1 | — | 1159RGM75 | TRUE |
| 3 | Divergência no valor do frete – Pedido #12345 | Divergência de Frete | Neutro | P3 | 12345 | — | FALSE |

**Análise dos resultados:**
- O **Teste 1** demonstra a regra "tom Insatisfeito → P2". O `Requer_Humano = TRUE` está correto porque dados críticos estão ausentes (sem pedido e sem viagem).
- O **Teste 2** demonstra a regra "tom Urgente/Irado → P1". O `Requer_Humano = TRUE` está correto por prioridade P1 **e** menção a risco regulatório (ANTT). O `Numero_Viagem` foi extraído corretamente.
- O **Teste 3** demonstra o caso normal: tom Neutro, divergência sem agravantes, `Requer_Humano = FALSE`. O `Numero_Pedido` e a `Transportadora` foram extraídos corretamente.

Os três testes cobrem **três faixas de prioridade diferentes** (P1, P2, P3), provando que o agente aplica as regras de negócio de forma consistente — não é apenas o tom que define a prioridade, mas a combinação de tom + valor + menção regulatória + presença de dados críticos.

## 3. Arquitetura da Solução

### 3.1 Visão Geral do Fluxo

O fluxo foi implementado no **n8n** (self-hosted) e é composto por **8 nós** conectados em sequência, com um nó de decisão (IF) que filtra as respostas automáticas do próprio sistema para evitar loops.

```text
┌─────────────────┐
│  Gmail Trigger  │  Monitora e-mails não lidos (poll a cada minuto)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│       IF        │  Filtra: remetente NÃO contém o e-mail da operação?
└───┬─────────┬───┘
    │ true    │ false
    ▼         ▼
┌─────────────┐  (sem conexão)
│ Message a   │  Gemini classifica e extrai JSON
│ model       │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│ Code in         │  Limpa markdown, faz parse do JSON
│ JavaScript      │  e normaliza os campos
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Append row in   │  Grava os dados na planilha
│ sheet           │  "Triagem Logística" → aba "Ocorrências"
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Code in         │  Monta o HTML da resposta
│ JavaScript1     │  personalizada (com quebras de linha)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Reply to a      │  Envia a resposta no mesmo thread
│ message         │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Mark a message  │  Marca o e-mail original como lido
│ as read         │  (evita reprocessamento)
└─────────────────┘
```

### 3.2 Descrição dos Nós

| # | Nó | Tipo | Função |
|---|----|------|--------|
| 1 | **Gmail Trigger** | Trigger | Monitora a caixa de entrada a cada minuto e dispara o fluxo quando há e-mail não lido (`readStatus: unread`). |
| 2 | **IF** | Condicional | Verifica se o remetente **não contém** o e-mail da operação (`ari.jvs2.0@gmail.com`). Se for a própria resposta automática, o fluxo para aqui (evita loop). Se for cliente/parceiro, segue para o Gemini. |
| 3 | **Message a model** | LLM (Gemini) | Recebe o prompt (System Prompt + dados do e-mail) e devolve o JSON estruturado com intenção, tom, prioridade e campos extraídos. Configurado com `temperature: 0.1` e `retryOnFail: true`. |
| 4 | **Code in JavaScript** | Code | Limpa a resposta do Gemini (remove markdown), faz o `JSON.parse`, trata erros de parse com fallback e **normaliza** os nomes dos campos (ex.: `intenção` → `intencao`, `numero_viagem` → `numero_viagem`). |
| 5 | **Append row in sheet** | Google Sheets | Adiciona uma nova linha na planilha `Triagem Logística` (aba `Ocorrências`), mapeando cada campo do JSON para a coluna correspondente. |
| 6 | **Code in JavaScript1** | Code | Monta o HTML da resposta automática com quebras de linha (`<br>`), personalizando com o nome do remetente, resumo e prioridade. |
| 7 | **Reply to a message** | Gmail | Envia a resposta no **mesmo thread** do e-mail original (`messageId` do Gmail Trigger), usando o HTML gerado pelo nó 6. |
| 8 | **Mark a message as read** | Gmail | Marca o e-mail original como **lido** (`markAsRead`), evitando que o Gmail Trigger o capture novamente na próxima verificação. |

### 3.3 Fluxo de Dados (Passo a Passo)

1. **Entrada:** um e-mail chega à caixa de entrada e fica com status "não lido".
2. **Detecção:** o `Gmail Trigger` (poll a cada minuto) identifica o e-mail e dispara o fluxo, passando `id`, `From`, `Subject` e `text`/`snippet`.
3. **Filtro:** o `IF` verifica o remetente. Se for a própria resposta automática (contém `ari.jvs2.0@gmail.com`), o fluxo para. Caso contrário, segue.
4. **Classificação:** o `Message a model` envia o prompt + dados do e-mail ao Gemini, que devolve um JSON com `intencao`, `tom`, `prioridade`, `numero_pedido`, `numero_viagem`, `transportadora`, `valor_divergencia`, `resumo`, `motivo` e `requer_humano`.
5. **Normalização:** o `Code in JavaScript` limpa a resposta (remove markdown, faz parse, trata erro) e normaliza os campos para o formato esperado pela planilha.
6. **Registro:** o `Append row in sheet` grava uma nova linha na planilha com data, remetente, assunto e todos os campos extraídos.
7. **Resposta:** o `Code in JavaScript1` monta o HTML personalizado, e o `Reply to a message` envia a resposta no mesmo thread.
8. **Encerramento:** o `Mark a message as read` marca o e-mail original como lido, impedindo que o trigger o processe novamente.

### 3.4 Decisões de Arquitetura

- **Filtro IF (nó 2):** sem ele, a resposta automática geraria um novo e-mail não lido no mesmo thread, e o fluxo entraria em loop infinito (o agente responderia a si mesmo).
- **Mark as Read (nó 8):** complementa o IF. Mesmo com o filtro, o Gmail pode reativar o thread como "não lido" após a resposta. O `Mark as Read` garante que o e-mail original não seja reprocessado.
- **Temperature 0.1:** escolhida para garantir consistência na classificação. Com temperature alta, o Gemini poderia variar a saída entre execuções (ex.: devolver `intenção` com acento em uma execução e sem acento em outra), quebrando o parser.
- **Retry on Fail:** ativado no nó Gemini para lidar com instabilidades temporárias da API do Google (erros 503), garantindo que o fluxo não pare por falhas transitórias.
- **Normalização no Code:** o Gemini pode devolver variações (com/sem acento, com/sem markdown). O nó Code trata essas variações antes de gravar na planilha, garantindo consistência nos dados.
- **HTML pré-montado no Code:** o n8n tem um bug conhecido ao renderizar HTML inline no campo `Message` do nó Gmail. A solução foi pré-montar o HTML em um nó Code e passar a string pronta via expressão (`{{ $json.htmlPronto }}`).

### 3.5 Stack Tecnológica

| Componente | Tecnologia |
|------------|------------|
| Orquestração | n8n (self-hosted, versão 2.35.7) |
| LLM | Google Gemini (`gemini-3.5-flash`) via API |
| E-mail | Gmail API (OAuth2) |
| Persistência | Google Sheets API (OAuth2) |
| Linguagem de script | JavaScript (nós Code do n8n) |
