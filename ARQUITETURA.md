# Arquitetura do jev-gw

Este documento explica **como a coisa é montada por dentro** e, principalmente, **por que
cada peça existe**. Se você vai incorporar o gateway num sistema seu, leia a seçêo 2 (o
caminho de uma decisêo) e a 5 (contrato de falha) — o resto é contexto.

---

## 1. O problema que o gateway resolve

Chamar o Jev direto do seu código funciona. O que nêo funciona bem é chamar o Jev **de
cinco lugares diferentes do seu sistema**: cada lugar reinventa o tratamento de erro, ninguém
sabe quanto se gastou no dia, um laço com bug consome crédito a noite inteira, e quando a API
cai o sistema cai junto.

O jev-gw é a **porta única**. Toda consulta passa por ela, e ela garante quatro coisas que
você nêo quer reescrever em cada ponto de integraçêo:

| Garantia | O que significa na prática |
|---|---|
| **Teto de gasto** | um valor em dólares por dia. Estourou, o gateway para de consultar |
| **Cache** | pedido idêntico dentro da validade nêo gasta de novo |
| **Falha conservadora** | Jev fora do ar, chave errada, resposta estranha → revisêo humana, nunca exceçêo |
| **Registro** | uma linha JSONL por chamada: custo, latência, decisêo, motivo |

---

## 2. O caminho de uma decisêo

```
        seu sistema
             │
             │  decidir(pedido)                 ← biblioteca (import)
             │  POST /decidir                   ← serviço (HTTP)
             │  python3 -m jev_gw decidir        ← terminal (CLI)
             ▼
   ┌─────────────────────────────────────────────────────────┐
   │  1. VALIDAÇêO      pedido respeita o contrato do Jev?    │
   │                    nêo → PedidoInvalido (erro de verdade)│
   ├─────────────────────────────────────────────────────────┤
   │  2. CACHE          impressêo digital SHA-256 do pedido   │
   │                    achou e está no prazo → devolve       │
   ├─────────────────────────────────────────────────────────┤
   │  3. ORÇAMENTO      gasto do dia < teto?                  │
   │                    nêo → devolve "review" (nêo consulta) │
   ├─────────────────────────────────────────────────────────┤
   │  4. CONSULTA       cliente Jev (TypeSafe ou OpenRouter)  │
   │                    3 tentativas, backoff, timeout        │
   ├─────────────────────────────────────────────────────────┤
   │  5. POLÍTICA       confiança e probabilidade nos limiares│
   │                    opçêo "insuficiente"? domínio sensível?│
   │                    → suggest  ou  review                 │
   ├─────────────────────────────────────────────────────────┤
   │  6. REGISTRO       uma linha JSONL: custo, latência, açêo │
   └─────────────────────────────────────────────────────────┘
             │
             ▼
   {acao, motivo, resposta, decisoes, custo_usd, latencia_ms, origem}
```

**A ordem importa e foi escolhida:**

- **Cache antes do orçamento.** Uma resposta que já está guardada nêo custa nada, entêo nêo
  faz sentido barrá-la por teto. Se fosse o contrário, um sistema com teto estourado
  perderia até as respostas que já tinha pago.
- **Orçamento antes da consulta.** Verificar depois seria verificar o prejuízo, nêo evitá-lo.
- **Política depois da resposta, nêo dentro do modelo.** Os limiares sêo do seu domínio, nêo
  do Jev. Triagem de e-mail e triagem clínica usam o mesmo modelo e limiares diferentes.

---

## 3. Os módulos

| Arquivo | Responsabilidade | Depende de |
|---|---|---|
| `jev_gw/nucleo.py` | orquestra os 6 passos acima. É a única porta | todos os outros |
| `jev_gw/config.py` | lê o ambiente, dá padrões utilizáveis | — |
| `jev_gw/cache.py` | impressêo digital, guarda e busca em arquivo | — |
| `jev_gw/orcamento.py` | teto diário; lê o gasto do próprio registro | `registro` |
| `jev_gw/registro.py` | JSONL append-only + resumo do dia | — |
| `jev_gw/servidor.py` | HTTP (`http.server`) e o painel | `nucleo`, `registro` |
| `jev_gw/__main__.py` | CLI | todos |
| `jev_gw/vendor/jev_core.py` | **cópia** do cliente Jev do Jev Decision Lab | — |

Nenhum módulo importa o `servidor` ou o `__main__`: as cascas dependem do núcleo, nunca o
contrário. É o que permite usar como biblioteca sem subir servidor nenhum.

### Três decisões de projeto que merecem explicaçêo

**O cliente Jev é vendorizado, nêo importado.** `jev_gw/vendor/jev_core.py` é uma cópia de
`jev_lab/core.py` (do repositório [inematds/jev](https://github.com/inematds/jev)), com o
sha256 da origem no cabeçalho. Um gateway pode rodar num servidor onde o repositório irmêo nêo
existe; depender dele seria uma dependência invisível que quebra na primeira máquina nova.
`scripts/sincronizar-core.py` refaz a cópia mostrando o diff e atualizando o hash.

**O orçamento lê o gasto do registro, nêo de um contador.** Contador separado significa dois
lugares para dessincronizar (e um a mais para corromper num desligamento). O custo é ler um
arquivo por consulta — irrelevante perto dos ~600 ms de uma chamada de rede.

**O cache fica em arquivo, nêo em memória.** Um sistema que importa a biblioteca dentro de um
processo curto (um worker, um job, um comando) perderia cache em memória toda execuçêo. Em
arquivo, todos os processos e o servidor compartilham o mesmo, e ele sobrevive a reinício.
A escrita usa troca atômica (`.tmp` + `replace`) para ninguém ler metade de um arquivo.

---

## 4. As três formas de incorporar

**Biblioteca** — sistema em Python. Nenhum processo extra, nenhuma porta.

```python
from jev_gw import decidir
saida = decidir(pedido)
```

**Serviço HTTP** — sistema em qualquer linguagem. Um processo, porta 8770 em `127.0.0.1`.

```bash
python3 -m jev_gw servir
curl -X POST localhost:8770/decidir -d @pedido.json
```

**CLI** — script, cron, pipeline. Código de saída: `0` sugestêo, `2` revisêo, `1` erro.

```bash
python3 -m jev_gw decidir pedido.json
```

As três chamam a mesma `nucleo.decidir()`. Nêo existe lógica só no servidor ou só na CLI.

---

## 5. Contrato de falha (a parte que importa na integraçêo)

**`decidir()` nêo levanta exceçêo por falha de infraestrutura.** Jev fora do ar, chave
ausente, timeout, resposta malformada, teto estourado — tudo devolve:

```json
{"acao": "review", "motivo": "...", "origem": "erro|bloqueado", "resposta": null}
```

O seu sistema trata `review` do jeito que já trata um caso que precisa de humano: fila,
regra antiga, resposta padrêo. **O gateway nunca é o motivo de uma requisiçêo sua falhar.**

A única exceçêo que sai daqui é `PedidoInvalido` — pedido fora do contrato do Jev (pergunta
sem tipo, contexto vazio, mais de 30 perguntas). Isso é erro de programaçêo e precisa
aparecer no desenvolvimento, nêo virar "revise" silencioso. No HTTP isso é **422**.

Os códigos HTTP seguem a mesma lógica:

| Código | Quando | O que fazer |
|---|---|---|
| **200** | decisêo tomada — inclusive `review` por falha do Jev | siga o `acao` |
| **400** | JSON quebrado, corpo vazio, opçêo inválida | conserte a chamada |
| **413** | pedido acima de 200 KB | reduza o contexto |
| **422** | pedido íntegro mas fora do contrato do Jev | conserte o pedido |
| **404** | rota errada | use `/decidir`, `/saude`, `/custo`, `/painel` |

Repare que **teto estourado é 200, nêo 429**: para quem chamou, nêo é um erro — é uma
decisêo de encaminhar para revisêo.

---

## 6. O que este gateway NêO faz

Dito de forma explícita, porque a palavra "gateway" é usada para muita coisa:

- **Nêo é proxy de LLM.** Ele nêo fica entre o seu agente e o GPT/Claude, nêo reescreve
  requisiçêo de chat, nêo intercepta escolha de ferramenta de agente de código. Quem faz
  isso é o [jev-gateway do vinilana](https://github.com/vinilana/jev-gateway) — projeto
  independente, em TypeScript, e um bom complemento a este.
- **Nêo executa ações.** Ele devolve uma sugestêo. Quem aplica, aplica com as próprias
  permissões. Nenhuma decisêo vira chamada de API, e-mail ou escrita em banco aqui dentro.
- **Nêo decide os seus limiares.** O padrêo (0,9 de confiança) é conservador de propósito.
  O número certo depende do custo de errar no seu domínio, e isso o gateway nêo sabe.
- **Nêo é um cache semântico.** A impressêo digital é do pedido exato. Pedido parecido é
  pedido diferente — e é assim que deve ser: contexto quase igual pode mudar a decisêo.
- **Nêo guarda o conteúdo dos pedidos.** O registro grava metadados (custo, latência, açêo,
  nomes das perguntas). O `state` — que é onde moram os dados do seu cliente — **nêo** é
  gravado. O cache guarda a resposta, num diretório só seu.

---

## 7. Onde ficam os dados

`~/.jev-gw/` por padrêo (`JEV_GW_DADOS` muda):

```
~/.jev-gw/
├── registro-2026-09-22.jsonl    uma linha por chamada, do dia
└── cache/
    └── <sha256 do pedido>.json  resposta guardada
```

Rotaçêo por dia acontece sozinha (o nome tem a data). Limpeza de cache é manual e explícita:
`python3 -m jev_gw cache --limpar --vencidos`.

---

## 8. Configuraçêo

Tudo por ambiente, tudo com padrêo que funciona:

| Variável | Padrêo | Para quê |
|---|---|---|
| `JEV_GW_TETO_DIARIO` | `1.00` | teto em dólares por dia. `0` desliga o gateway |
| `JEV_GW_TTL_CACHE` | `900` | segundos que uma resposta vale |
| `JEV_GW_TIMEOUT` | `5.0` | segundos por consulta |
| `JEV_GW_PORTA` | `8770` | porta do serviço |
| `JEV_GW_HOST` | `127.0.0.1` | **mude com cuidado**: o gateway guarda a sua chave |
| `JEV_GW_DADOS` | `~/.jev-gw` | onde ficam registro e cache |
| `JEV_PROVIDER` | `typesafe` | `typesafe` ou `openrouter` |
| `TYPESAFE_API_KEY` / `OPENROUTER_API_KEY` | — | a chave, conforme o provedor |

A porta 8770 foi escolhida fora da faixa 8788–8791 usada pelo jev-gateway do vinilana, para
os dois poderem rodar na mesma máquina.
