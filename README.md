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
