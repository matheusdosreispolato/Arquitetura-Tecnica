# Engenharia de Prompts

> Um prompt é a interface entre a sua intenção e o modelo. Este documento cobre os conceitos fundamentais, técnicas práticas e como economizar tokens sem perder qualidade.

---

## Definições Essenciais

| Termo | Definição |
|---|---|
| **Prompt** | O texto completo enviado ao modelo (system + histórico + mensagem atual) |
| **System prompt** | Instrução persistente que define comportamento, papel e restrições do modelo |
| **Token** | Unidade de texto processada pelo modelo (~0,75 palavras em inglês; ~0,6 em português) |
| **Context window** | Limite máximo de tokens que o modelo consegue processar de uma vez (entrada + saída) |
| **Temperature** | Controla aleatoriedade: `0.0` = determinístico, `1.0` = criativo/variável |
| **Completion** | A resposta gerada pelo modelo |
| **Few-shot** | Técnica de fornecer exemplos de entrada/saída dentro do prompt |
| **Chain-of-thought** | Técnica de pedir que o modelo "pense em voz alta" antes de responder |
| **Grounding** | Ancorar a resposta em um contexto fornecido, reduzindo alucinação |
| **Hallucination** | Quando o modelo inventa fatos com aparência de verdade |

---

## Estrutura de uma Mensagem

A maioria dos modelos modernos usa três papéis:

```
system   → comportamento global, papel, restrições (configurado uma vez)
user     → a pergunta ou tarefa atual
assistant → a resposta do modelo (ou um prefixo para guiar o formato)
```

**Exemplo:**
```python
messages = [
    {
        "role": "system",
        "content": "Você é um analista de dados. Responda em português. Seja objetivo."
    },
    {
        "role": "user",
        "content": "Qual foi o produto mais vendido no último trimestre?"
    }
]
```

---

## Como Criar um Prompt Eficaz

### 1. Defina o papel (Role)

Quem o modelo deve "ser" nesta tarefa:

```
# ❌ Genérico
"Analise este contrato."

# ✅ Com papel definido
"Você é um advogado especializado em contratos de locação comercial no Brasil.
Analise o contrato abaixo e identifique cláusulas que violem a lei ou que
coloquem o locatário em desvantagem."
```

### 2. Dê contexto suficiente

O modelo não sabe nada além do que você enviou:

```
# ❌ Sem contexto
"O pedido foi aprovado?"

# ✅ Com contexto
"Temos um processo de aprovação com três critérios:
(1) valor abaixo de R$10.000,
(2) cliente sem inadimplência,
(3) produto em estoque.

Dado o pedido abaixo, ele foi aprovado?
[dados do pedido]"
```

### 3. Especifique o formato de saída

Modelos sem instrução de formato produzem texto livre — difícil de processar:

```
# ❌ Sem formato
"Liste os riscos deste projeto."

# ✅ Com formato
"Liste os riscos deste projeto em JSON:
[{"risco": str, "probabilidade": "alta|media|baixa", "impacto": "alto|medio|baixo"}]
Retorne APENAS o JSON, sem texto adicional."
```

### 4. Use exemplos (Few-Shot)

Para tarefas de classificação, extração ou formatação, exemplos valem mais que instruções:

```
Classifique o sentimento dos textos como POSITIVO, NEGATIVO ou NEUTRO.

Exemplos:
Texto: "Adorei o atendimento!"
Sentimento: POSITIVO

Texto: "Chegou errado e o suporte sumiu."
Sentimento: NEGATIVO

Texto: "O pedido foi entregue no prazo."
Sentimento: NEUTRO

---
Texto: "A entrega foi rápida, mas o produto não era o que esperava."
Sentimento:
```

> Inclua exemplos dos casos **difíceis e ambíguos** — os óbvios o modelo já acerta sem exemplo.

### 5. Peça raciocínio explícito (Chain-of-Thought)

Para tarefas que exigem múltiplas etapas:

```
Antes de dar sua resposta final, liste os critérios que você verificou
e o que encontrou em cada um.
```

---

## Economia de Tokens

Tokens custam dinheiro e ocupam a janela de contexto. Cada token que você elimina sem perder clareza é um ganho.

### Técnicas para reduzir tokens de entrada

**1. Remova palavras de cortesia e rodeios**
```
# ❌ 22 tokens
"Olá! Por favor, poderia me ajudar a analisar o texto abaixo e identificar..."

# ✅ 7 tokens
"Analise o texto abaixo e identifique..."
```

**2. Use delimitadores em vez de frases longas**
```
# ❌ 18 tokens
"O texto que você deve resumir começa agora e termina quando eu disser FIM:"

# ✅ 5 tokens
"Resuma o texto entre <texto> e </texto>:
<texto>
...
</texto>"
```

**3. Troque listas de regras por tabelas**
```
# ❌ Prolixo
"Se o status for PENDENTE, não faça nada. Se o status for PAGO, envie confirmação.
Se o status for CANCELADO, acione o reembolso."

# ✅ Conciso
"Ações por status:
PENDENTE → ignorar
PAGO → enviar confirmação
CANCELADO → acionar reembolso"
```

**4. Truncar contexto longo**

Se você tem um documento de 10 páginas mas a pergunta se refere apenas a uma seção, envie apenas essa seção:
```python
# Extrair apenas o trecho relevante antes de enviar
trecho_relevante = extrair_secao(documento, "Cláusulas de rescisão")
prompt = f"Analise as cláusulas de rescisão:\n\n{trecho_relevante}"
```

**5. Evitar repetição de contexto**

Em conversas longas, não repita o contexto inteiro a cada mensagem. Mantenha o histórico na API — o modelo já tem acesso a ele.

### Técnicas para reduzir tokens de saída

**1. Especifique o tamanho esperado**
```
"Responda em no máximo 3 bullets."
"Resuma em 2 frases."
"Retorne apenas o JSON, sem explicação."
```

**2. Use `max_tokens` como proteção**
```python
client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=256,    # limite da resposta — não deixe em aberto
    ...
)
```

**3. Evite pedir explicações quando não precisa**
```
# ❌ Pede texto + JSON (tokens dobrados)
"Explique sua análise e retorne os dados em JSON."

# ✅ Quando só o dado importa
"Retorne apenas o JSON com os dados extraídos."
```

### Escolha o modelo certo para cada tarefa

| Tarefa | Modelo recomendado | Por quê |
|---|---|---|
| Classificação, extração simples | Haiku / GPT-4o-mini | Rápido, barato, preciso o suficiente |
| Análise complexa, raciocínio | Sonnet / GPT-4o | Equilíbrio qualidade/custo |
| Revisão jurídica, código crítico | Opus / GPT-4o | Máxima qualidade quando justificado |
| Eval de respostas em lote | Haiku | Custo mínimo para avaliação automatizada |

---

## Templates de System Prompt

### Para análise de dados

```
Você é um analista de dados sênior.
Responda sempre em português do Brasil.
Seja direto: vá ao ponto antes de explicar o raciocínio.
Se não tiver informação suficiente para responder, diga explicitamente o que está faltando.
Nunca invente dados — cite apenas o que foi fornecido no contexto.
```

### Para geração de código

```
Você é um engenheiro Python sênior.
Siga PEP 8 e use type hints em todas as funções.
Responda apenas com código funcional — sem markdown desnecessário.
Inclua docstrings no padrão Google.
Se houver mais de uma forma de resolver, apresente a mais simples primeiro.
```

### Para extração estruturada

```
Você extrai dados de textos não estruturados.
Responda SEMPRE com JSON válido no schema fornecido.
Não inclua texto fora do JSON.
Se um campo não estiver no texto, use null.
Não invente dados ausentes.
```

---

## Armadilhas Comuns

| Problema | Sintoma | Solução |
|---|---|---|
| Prompt vago | Respostas inconsistentes a cada execução | Adicionar contexto e definir formato de saída |
| Tarefa composta | Modelo faz metade de cada coisa | Uma chamada por tarefa |
| Instrução negativa sem positiva | Modelo ignora a restrição | Dizer o que fazer, não apenas o que não fazer |
| Temperature alta em extração | Dados diferentes a cada execução | `temperature=0.0` para tarefas determinísticas |
| Confiar no JSON sem validar | `JSONDecodeError` em produção | Sempre validar com Pydantic + tratar exceção |
| System prompt longo demais | Alto custo fixo em toda chamada | Manter system prompt enxuto; colocar contexto variável na mensagem |
| Repetir instrução no user e no system | Tokens dobrados sem benefício | Instrução vai no system; dados variáveis vão no user |

---

## Iteração de Prompts

Um prompt raramente fica bom na primeira versão. O processo correto:

```
1. Escrever o prompt inicial
2. Testar com 5–10 casos representativos (incluindo os difíceis)
3. Anotar onde falha: formato errado? dado incorreto? recusa?
4. Fazer uma mudança por vez — não reescrever tudo de uma vez
5. Retestar incluindo os casos anteriores (evitar regressão)
6. Repetir até atingir a qualidade esperada
```

> Para estruturar avaliações formais de prompts, consulte `IA/04-evals.md`.
