# Precifica.AI

Assistente de precificação para trabalhadores autônomos. A pessoa conversa em linguagem natural ("cobro R$ 150, gasto R$ 21,50 de material e levo 2h30") e o assistente mostra quanto realmente sobra, quanto ela ganha por hora e qual é o preço mínimo para não ter prejuízo ou para atingir uma meta.

**Demonstração online:** https://precifica-ai.onrender.com

**Princípio central:** a IA (Google Gemini) **só interpreta o texto** e devolve os dados em JSON. **Todas as contas são feitas em Python**, de forma determinística e com precisão decimal. A IA nunca calcula.

---

## Sumário

1. [Funcionalidades](#1-funcionalidades)
2. [Como funciona](#2-como-funciona)
3. [Regras de cálculo](#3-regras-de-cálculo)
4. [Estrutura do projeto](#4-estrutura-do-projeto)
5. [Instalação e execução local](#5-instalação-e-execução-local)
6. [Configuração (.env)](#6-configuração-env)
7. [API HTTP](#7-api-http)
8. [Saída do motor (`engine.py`)](#8-saída-do-motor-enginepy)
9. [Front-end](#9-front-end)
10. [Testes pelo terminal](#10-testes-pelo-terminal)
11. [Deploy no Render](#11-deploy-no-render)
12. [Limites da API e solução de problemas](#12-limites-da-api-e-solução-de-problemas)
13. [Uso responsável da IA](#13-uso-responsável-da-ia)
14. [Limitações conhecidas](#14-limitações-conhecidas)

---

## 1. Funcionalidades

- **Conversa em linguagem natural:** entende formatos como "mil e duzentos", "1.200,50", "1h30 + 30 min de preparo" e "rende uns 40 atendimentos".
- **Dois modelos de cobrança:** por serviço (maquiagem, manicure, costura, artesanato...) e recorrente (plano mensal, assinatura, aulas, hospedagem).
- **Resultado em linguagem simples:** custos detalhados, sobra, valor por hora real e preço mínimo.
- **Meta opcional:** com "quero ganhar R$ 50 por hora" ou "quero ganhar R$ 5.000 por mês", o assistente calcula o preço para atingir a meta.
- **Simulações:** "E se eu cobrar R$ 60?" compara os dois cenários lado a lado, com a taxa recalculada. No modelo recorrente, também "E se eu tiver 20 clientes?".
- **Embalagens e rendimento:** "caixa com 10 pares por R$ 35, uso 1 por cliente" vira R$ 3,50 por atendimento.
- **Custos mensais rateados:** aluguel, imposto, agenda e outros gastos fixos são divididos pelos atendimentos ou clientes do mês.
- **Confirmação antes de calcular:** o assistente mostra os dados que entendeu e só calcula após o "sim".
- **Correções no meio da conversa:** "na verdade a base rende só 30" recalcula tudo.
- **Validação de entradas:** valores negativos, tempo zero e preço zero são recusados com uma pergunta, e o assistente pergunta o que falta em vez de inventar.
- **Estimativas sinalizadas:** valores aproximados ("uns R$ 40") aparecem como "(estimativa sua)".

---

## 2. Como funciona

```
Mensagem do usuário
   │
   ▼
extrair_dados_com_ia()      Gemini: texto → JSON (nunca calcula)
   │
   ▼
normalizar_extracao()       Python: valida tipos, recusa negativos/zeros, soma tempos em partes
   │
   ▼
atualizar_estado()          Python: mescla com o que já foi dito na conversa
   │
   ├─ falta dado essencial ──────► pergunta o que falta
   ├─ dados completos (1ª vez) ──► mostra o resumo e pede confirmação
   └─ confirmado ────────────────► calcula (por serviço ou recorrente) e explica
```

### Fluxo típico de conversa

1. **Pessoa:** "Faço maquiagem social e cobro R$ 150."
2. **Assistente:** registra o preço e pergunta tempo (com preparo e limpeza), materiais, deslocamento e forma de pagamento.
3. **Pessoa:** responde, em uma ou várias mensagens.
4. **Assistente:** mostra o resumo dos dados e pergunta "Está tudo certo?".
5. **Pessoa:** "sim". Esse "sim" é reconhecido sem chamar a IA.
6. **Assistente:** mostra o resultado:

```
Resumo do seu serviço (Maquiagem social):
- Descartáveis: R$ 3,50
- Produtos: R$ 10,00
- Cílios: R$ 8,00
- Total de materiais: R$ 21,50
- Deslocamento: R$ 24,00
- Taxa do cartão (3,5% de R$ 150,00): R$ 5,25
- Total de custos: R$ 50,75

Cobrando R$ 150,00:
- Sobra para você: R$ 99,25
- Valor por hora: R$ 39,70 (2,5h de trabalho, contando preparo e limpeza)

💡 Preço mínimo: R$ 47,15. Abaixo disso você não cobre nem os custos.
```

### Intenções reconhecidas

| Intenção | Exemplo | O que acontece |
|---|---|---|
| `informar_dados` | "Gasto R$ 15 de material" | Adiciona os dados à conversa |
| `corrigir_dado` | "Na verdade o tecido custou R$ 35" | Altera só o item citado |
| `simular_preco` | "E se eu cobrar R$ 90?" | Compara os cenários, sem alterar o preço salvo |
| `confirmar` | "Sim, pode calcular" | Libera o cálculo |

### Dados obrigatórios

O assistente não calcula enquanto faltar um destes dados:

| Modelo | Obrigatórios | Perguntados, mas não bloqueiam |
|---|---|---|
| Por serviço | preço, tempo, materiais, deslocamento e quanto rende cada embalagem citada | forma de pagamento |
| Recorrente | nº de clientes, preço (ou meta de renda), custos mensais e nº de encontros (se houver custo por encontro) | forma de pagamento e material por cliente |

"Não tenho deslocamento", "não gasto material" e "nenhum gasto mensal" contam como respostas válidas, com valor zero. Se a forma de pagamento não for informada, o cálculo usa **5% de taxa de cartão** e **avisa isso na resposta**.

---

## 3. Regras de cálculo

Todas as contas usam `Decimal`, com arredondamento em centavos (`ROUND_HALF_UP`).

### Por serviço

| Item | Fórmula |
|---|---|
| Material por atendimento | `preço da embalagem ÷ rendimento × quantidade usada` |
| Custos mensais rateados | `custos fixos do mês ÷ atendimentos por mês` |
| Taxa | `preço × taxa (%)`, recalculada quando o preço muda |
| **Total de custos** | `materiais + deslocamento + outros + rateio + taxa` |
| **Sobra** | `preço − total de custos` |
| **Valor por hora** | `sobra ÷ horas (atendimento + preparo + limpeza)` |
| Preço mínimo, sem meta | `custos sem taxa ÷ (1 − taxa)` |
| Preço mínimo, com meta | `(custos sem taxa + horas × meta por hora) ÷ (1 − taxa)` |

### Recorrente (base mensal)

| Item | Fórmula |
|---|---|
| Conversão de períodos | anual ÷ 12, semanal × 52/12, diário × 30 e gasto único diluído em 12 meses |
| Horas "por dia" | × 22 dias de trabalho (configurável) |
| Material por cliente | custo por cliente/mês, ou custo por encontro × encontros por mês |
| Rateio por cliente | `custos fixos do mês ÷ clientes` |
| **Sobra por cliente** | `preço mensal − (material + rateio + taxa)` |
| **Valor por hora** | `sobra por cliente ÷ (encontros × horas por encontro)`, ou sobra total ÷ horas totais |
| Preço mínimo | `(custos fixos + material de todos os clientes + meta de renda) ÷ (clientes × (1 − taxa))` |
| Clientes necessários | quantos clientes, no preço atual, cobrem os custos e a meta |

Quando o assistente adota uma premissa (por exemplo, "considerei o desenvolvimento como gasto mensal" ou "22 dias de trabalho por mês"), ela aparece na resposta.

---

## 4. Estrutura do projeto

```
├── api.py                # FastAPI: rota POST /chat e página do chat em GET /
├── engine.py             # Motor: integração com o Gemini + validação + cálculo + textos
├── frontend/
│   ├── index.html        # Interface de chat
│   ├── chat.js           # Envio das mensagens, estado da conversa e renderização segura
│   └── style.css
├── test_engine.py        # Testes unitários (78) e de integração com o Gemini (4, opcionais)
├── avaliar_casos.py      # Avaliador dos 5 cenários de referência (gera relatorio_casos.md)
├── chat_terminal.py      # Conversa com o assistente direto no terminal
├── requirements.txt      # google-genai, python-dotenv, fastapi, uvicorn
├── .env.example          # Modelo do arquivo de configuração
└── .gitignore            # Inclui .env
```

---

## 5. Instalação e execução local

**Pré-requisitos:** Python 3.10 ou superior e uma chave gratuita da API do Gemini (https://aistudio.google.com/apikey).

```bash
git clone https://github.com/otavio-pierin/Precifica.AI.git
cd Precifica.AI
pip install -r requirements.txt
```

Crie o arquivo `.env` na raiz do projeto com a sua chave:

```
GEMINI_API_KEY=sua_chave_aqui
```

> No PowerShell, use `[IO.File]::WriteAllText("$PWD\.env", "GEMINI_API_KEY=sua_chave_aqui")` ou crie o arquivo pelo Bloco de Notas. **Nunca faça commit do `.env`.**

Suba o servidor:

```bash
uvicorn api:app --reload
```

Depois, abra **http://localhost:8000**.

---

## 6. Configuração (.env)

Só a `GEMINI_API_KEY` é obrigatória. As demais variáveis já têm os valores recomendados como padrão.

| Variável | Padrão | Para que serve |
|---|---|---|
| `GEMINI_API_KEY` | — | Chave da API do Gemini (**obrigatória**) |
| `GEMINI_MODEL` | `gemini-3.5-flash-lite` | Modelo principal |
| `GEMINI_MODELOS_RESERVA` | `gemini-3.8-flash` | Modelos tentados se o principal falhar (separados por vírgula; um espaço desativa) |
| `GEMINI_THINKING` | `minimal` | Nível de raciocínio nos modelos Gemini 3+ (`minimal`, `low`, `medium`, `high`) |
| `GEMINI_TIMEOUT` | `25` | Tempo máximo de cada chamada, em segundos |
| `GEMINI_PRAZO_TOTAL` | `40` | Tempo máximo somando todas as tentativas, em segundos |
| `GEMINI_TENTATIVAS` | `2` | Tentativas por modelo em erros 500/502/503 |
| `GEMINI_ESPERA` | `1.5` | Espera inicial entre tentativas (backoff), em segundos |
| `PEDIR_CONFIRMACAO` | `1` | `1` mostra os dados e espera "sim" antes do primeiro cálculo; `0` calcula direto |
| `DIAS_TRABALHO_MES` | `22` | Dias usados para converter "horas por dia" em horas por mês |

Variáveis definidas no terminal têm prioridade sobre o `.env`, o que é útil para testes pontuais:

```powershell
$env:GEMINI_TIMEOUT = "40"   # vale só para esta janela do terminal
```

---

## 7. API HTTP

### `GET /`
Devolve a interface de chat (`frontend/index.html`). Os arquivos estáticos ficam em `/static`.

### `POST /chat`

**Entrada:**

```json
{
  "mensagem": "Faço maquiagem social e cobro R$ 150.",
  "estado": null
}
```

Envie `estado: null` para começar uma conversa e, nas mensagens seguintes, o `estado` devolvido pela resposta anterior.

**Saída:**

```json
{
  "resposta": "Anotei: preço R$ 150,00.\n\nPara calcular, ainda preciso saber: ...",
  "estado": { "...": "dados acumulados da conversa" },
  "sucesso": true,
  "intencao": "informar_dados",
  "metricas": null,
  "simulacao": null,
  "informacoes_faltantes": ["tempo_horas", "materiais", "deslocamento"]
}
```

**Testando pelo terminal** (troque a URL por `http://localhost:8000` para testar localmente):

```bash
# Linux / macOS / Git Bash
curl -X POST https://precifica-ai.onrender.com/chat \
  -H "Content-Type: application/json" \
  -d '{"mensagem": "Faço maquiagem social e cobro R$ 150.", "estado": null}'
```

```powershell
# PowerShell: conversa em duas mensagens, reaproveitando o estado
$url = "https://precifica-ai.onrender.com/chat"
$r1 = Invoke-RestMethod $url -Method Post -ContentType "application/json; charset=utf-8" `
      -Body (@{ mensagem = "Cobro R$ 80 na maquiagem, gasto R$ 15 de material, R$ 10 de Uber, levo 2h e recebo no Pix"; estado = $null } | ConvertTo-Json)
$r1.resposta

$r2 = Invoke-RestMethod $url -Method Post -ContentType "application/json; charset=utf-8" `
      -Body (@{ mensagem = "sim"; estado = $r1.estado } | ConvertTo-Json -Depth 10)
$r2.resposta
```

---

## 8. Saída do motor (`engine.py`)

A função principal é `processar_mensagem_usuario(mensagem, estado_atual=None)`. Ela **sempre** devolve as mesmas chaves, inclusive em caso de erro:

| Chave | Descrição | Mostrar ao usuário? |
|---|---|---|
| `sucesso` | `False` só se a IA falhou ou um valor era inválido | Não |
| `intencao` | `informar_dados`, `corrigir_dado`, `simular_preco`, `confirmar` ou `null` | Não |
| `dados_extraidos` | JSON normalizado vindo da IA | Não (depuração) |
| `estado` | Dados acumulados da conversa | Não, mas **guarde e reenvie** |
| `metricas` | Números do cálculo (quando houve cálculo) | Opcional |
| `simulacao` | Comparação de cenários (em `simular_preco`) | Opcional |
| `informacoes_faltantes` | Dados essenciais que ainda faltam | Opcional |
| `resposta` | Texto pronto, em markdown simples | **Sim, sempre** |
| `erro` | Detalhe técnico | Não (só log) |

Principais campos de `metricas`:

- **Por serviço:** `total_materiais`, `materiais_detalhe`, `deslocamento`, `valor_taxa`, `custos_fixos_rateados`, `custos_totais`, `sobra_real`, `valor_hora_real`, `preco_minimo`, `tem_meta`, `margem_pct`.
- **Recorrente:** `custos_fixos_mensais`, `rateio_por_cliente`, `custo_por_cliente_mensal`, `custos_por_cliente`, `sobra_por_cliente`, `sobra_real` (total do mês), `valor_hora_real`, `preco_minimo`, `preco_minimo_periodo`, `clientes_necessarios`, `falta_para_meta`, `premissas`.

> Por compatibilidade, `custo_total` e `sobra_liquida` continuam existindo no conceito antigo, que desconta a meta de ganho. Para o resultado exibido ao usuário, use `custos_totais`, `sobra_real` e `valor_hora_real`.

O `estado` é um JSON simples. Além dos dados informados, ele guarda três campos de controle: `estimados` (campos que o usuário estimou), `aguardando_confirmacao` e `confirmado`.

---

## 9. Front-end

A interface (`frontend/`) é HTML, CSS e JavaScript puro, com visual de aplicativo de mensagens:

- **Contrato com a API:** envia `{ mensagem, estado }` para `/chat`, mostra `resposta` e guarda o `estado` devolvido, inclusive quando `sucesso` é `false`.
- **Conversa salva no navegador:** a conversa fica no `sessionStorage`, então sobrevive a um F5, e o botão **Nova conversa** a reinicia.
- **Renderização segura:** o texto da resposta é montado sem `innerHTML` (negrito, listas e parágrafos), o que evita injeção de código.
- **Uma mensagem por vez:** o envio fica bloqueado enquanto a resposta não chega, e respostas atrasadas de uma conversa já reiniciada são descartadas.

---

## 10. Testes pelo terminal

### Testes unitários (não usam a API nem gastam cota)

```bash
python -m unittest -v test_engine
```

São 78 testes locais. Eles cobrem:
- o motor financeiro;
- a validação das respostas da IA;
- a mescla de estado e a confirmação;
- os dois modelos de cobrança;
- a resiliência a falhas da API;
- os 5 cenários do gabarito.

### Testes de integração com o Gemini real (gastam 4 requisições)

```powershell
$env:RUN_INTEGRATION = "1"
python -m unittest -v test_engine.TestIntegracaoGemini
```

### Avaliador dos 5 cenários de referência

Compara as respostas do assistente com um gabarito calculado à mão: caminho feliz, comparação de preços, pacote × unidade, dados inválidos e plano mensal. Gera o `relatorio_casos.md` só com o que não bateu.

```bash
python avaliar_casos.py                      # offline: extração simulada, sem gastar cota
python avaliar_casos.py --ia                 # com o Gemini real (~35 requisições)
python avaliar_casos.py --ia --cenarios 1,3  # só os cenários escolhidos
```

### Conversa interativa no terminal

```bash
python chat_terminal.py           # digite "nova" para recomeçar e "sair" para encerrar
python chat_terminal.py --debug   # mostra também intenção, dados extraídos e estado
```

### Diagnóstico rápido da API (tempo e modelo que respondeu)

```bash
python -c "import logging,time,engine; logging.basicConfig(level=logging.WARNING); t=time.time(); r=engine.processar_mensagem_usuario('Cobro 60 na unha', None); print(round(time.time()-t,1),'s |', r['sucesso'], r['erro'])"
```

---

## 11. Deploy no Render

A aplicação está publicada como **Web Service** no Render: https://precifica-ai.onrender.com

| Configuração | Valor |
|---|---|
| Runtime | Python |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `uvicorn api:app --host 0.0.0.0 --port $PORT` |
| Variáveis de ambiente | `GEMINI_API_KEY` (obrigatória) e, se quiser, as outras da seção 6 |

Configure a chave em **Environment** no painel do Render, nunca no código. No plano gratuito, o serviço "dorme" depois de um tempo sem acessos, e o primeiro acesso seguinte pode levar até cerca de um minuto para responder.

---

## 12. Limites da API e solução de problemas

No plano gratuito, cada modelo tem limites de requisições por minuto e por dia, que podem ser consultados no AI Studio. Cada mensagem do chat consome uma requisição. O "sim" de confirmação não consome nenhuma.

| Sintoma no log | Causa | O que fazer |
|---|---|---|
| `429` | Cota do modelo esgotada | Esperar a cota reiniciar ou trocar a `GEMINI_API_KEY` por uma chave de outra conta |
| `504` | Modelo lento (o prazo estourou) | Tentar mais tarde ou aumentar `GEMINI_TIMEOUT` |
| `503` | Sobrecarga do Google | Tentar mais tarde (o código já repete e troca de modelo) |
| `400 ... Thinking level ... not supported` | O modelo não aceita o nível de raciocínio | O engine passa sozinho a usar `low` nesse modelo |
| `GEMINI_API_KEY não configurada` | `.env` ausente, mal escrito ou salvo com BOM | Recriar o `.env` (o engine já tolera BOM) |
| Aviso `LF will be replaced by CRLF` no Git | Quebra de linha no Windows | Inofensivo |

Quando nenhum modelo responde dentro do prazo, o usuário recebe uma mensagem amigável e a conversa é preservada.

---

## 13. Uso responsável da IA

- **Contas fora da IA:** a IA só interpreta texto, e todas as contas são feitas em código testado.
- **Supervisão humana:** o usuário confirma os dados antes do cálculo e pode corrigir qualquer valor.
- **Transparência:** premissas e estimativas são sempre exibidas, e nada é preenchido sem aviso.
- **Privacidade:** não pedimos dados pessoais, o servidor não armazena conversas (o estado fica no navegador) e a chave da API fica fora do código.
- **Eficiência:** usamos o modelo mais leve, raciocínio mínimo, no máximo uma chamada por mensagem e confirmações sem chamada à IA.
- **Plano gratuito:** nos termos do plano gratuito do Gemini, o Google pode usar o conteúdo enviado para melhorar seus produtos. Para uso em produção, recomenda-se um plano pago com garantia de não retenção.

---

## 14. Limitações conhecidas

- **Um serviço por conversa:** para precificar outro serviço, use **Nova conversa**.
- **Remoção de itens:** não há como remover um item, mas dá para corrigir o valor para zero ("o transporte foi 0").
- **Depende da IA para interpretar:** frases muito ambíguas podem ser mal interpretadas. A etapa de confirmação existe para o usuário conferir antes do cálculo.
- **Sem histórico entre sessões:** não há login nem histórico salvo de uma visita para outra.
- **Plano gratuito:** o Render e a API do Gemini no plano gratuito têm limites de uso e podem ficar lentos em horários de pico.