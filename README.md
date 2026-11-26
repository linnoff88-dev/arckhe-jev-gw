ï»¿# ê°Å¸Å¡Âª jev-gw ê¢â¬â gateway de decisêÂ£o para o Jev

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="48" />
  <p><strong>Arckhé AI</strong></p>
</div>


**ê°Å¸â¡Â§ê°Å¸â¡Â· [PortuguêÂªs](README.md) êÂ· ê°Å¸â¡Âºê°Å¸â¡Â¸ [English](README.en.md) êÂ· ê°Å¸â¡Âªê°Å¸â¡Â¸ [EspaêÂ±ol](README.es.md)**

[![jev-gw](guia/assets/banner.jpg)](https://linnoff88-dev.github.io/arckhe-/guia/)

## ê°Å¸ââ Guia de uso

Guia completo (landing + passo a passo): **https://linnoff88-dev.github.io/arckhe-/guia/**

Porta êÂºnica para toda consulta ao [Jev](https://docs.typesafe.ai/introduction) de um sistema:
valida o pedido, respeita um **teto de gasto diêÂ¡rio**, aproveita **cache**, aplica a **polêÂ­tica
de abstenêÂ§êÂ£o** e **registra custo e latêÂªncia** de cada chamada.

E o principal: **quando algo falha, devolve revisêÂ£o humana em vez de quebrar quem chamou.**
Jev fora do ar, chave errada, timeout, teto estourado ê¢â¬â o seu sistema continua de pêÂ©.

SêÂ³ biblioteca padrêÂ£o do Python 3.10+. **Nada para instalar.**

> **Como funciona por dentro: [ARQUITETURA.md](ARQUITETURA.md)** ê¢â¬â o caminho de uma decisêÂ£o,
> por que a ordem das etapas êÂ© essa, o contrato de falha e o que o gateway nêÂ£o faz.

## Incorporar no seu sistema

**Python ê¢â¬â importa e usa.** Sem processo extra, sem porta.

```python
from jev_gw import decidir

saida = decidir({
    'model': 'jev-1.13.0',
    'state': 'Mensagem do cliente: comprei ontem e quero devolver, nem abri a caixa.',
    'questions': {
        'fila': {
            'type': 'choice',
            'instructions': 'Para qual fila encaminhar esta mensagem?',
            'criteria': {
                'reembolso': 'Pedido de devoluêÂ§êÂ£o ou estorno',
                'suporte': 'DêÂºvida de uso ou defeito',
                'insuficiente': 'NêÂ£o dêÂ¡ para decidir com o que estêÂ¡ escrito',
            },
        },
    },
})

if saida['acao'] == 'suggest':
    encaminhar(saida['resposta']['answers']['fila']['choice'])
else:
    fila_de_revisao(saida['motivo'])       # inclui "o Jev caiu" ê¢â¬â e estêÂ¡ tudo bem
```

**Qualquer outra linguagem ê¢â¬â um POST.**

```bash
python3 -m jev_gw servir          # 127.0.0.1:8770
curl -X POST localhost:8770/decidir -H 'content-type: application/json' -d @pedido.json
```

**Terminal, cron, pipeline.** SaêÂ­da `0` = sugestêÂ£o, `2` = precisa de revisêÂ£o.

```bash
python3 -m jev_gw decidir pedido.json
```

Exemplos rodando, um por forma: [`exemplos/`](exemplos/) (Python, `curl`, Node ê¢â¬â todos sem
dependêÂªncia).

## Instalar

```bash
git clone https://github.com/linnoff88-dev/jev-gw.git
cd jev-gw
python3 -m unittest discover -s testes     # 23 testes, nenhum gasta crêÂ©dito
```

Para usar de outro projeto, aponte o `PYTHONPATH` para a pasta, ou copie o diretêÂ³rio
`jev_gw/` para dentro do seu projeto ê¢â¬â êÂ© autocontido de propêÂ³sito.

A chave sêÂ³ êÂ© necessêÂ¡ria na primeira consulta real:

```bash
export TYPESAFE_API_KEY=...        # ou OPENROUTER_API_KEY + JEV_PROVIDER=openrouter
```

## O que vocêÂª ganha

| | |
|---|---|
| **Teto de gasto** | `JEV_GW_TETO_DIARIO=1.00`. Estourou, vira revisêÂ£o ê¢â¬â nêÂ£o consulta mais |
| **Cache** | pedido idêÂªntico em 15 min nêÂ£o gasta de novo (medido: 610 ms ê¢â â 0 ms, US$ 0) |
| **Falha conservadora** | nenhuma falha de infraestrutura sobe como exceêÂ§êÂ£o para vocêÂª |
| **Registro** | JSONL com custo, latêÂªncia e decisêÂ£o por chamada |
| **Painel** | `http://127.0.0.1:8770/painel` ê¢â¬â gasto do dia, latêÂªncia, êÂºltimas chamadas |
| **Privacidade** | o `state` (seus dados) **nêÂ£o** vai para o registro. Escuta sêÂ³ em `127.0.0.1` |

## Comandos

```bash
python3 -m jev_gw servir                  # sobe o serviêÂ§o + painel
python3 -m jev_gw decidir pedido.json     # uma decisêÂ£o
python3 -m jev_gw saude                   # estado e teto, sem gastar nada
python3 -m jev_gw custo --dia 2026-09-22  # resumo do dia
python3 -m jev_gw cache --limpar          # manutenêÂ§êÂ£o
```

## Testado de verdade (22/09/2026)

- **23 testes** passando, todos com avaliador controlado ê¢â¬â a suêÂ­te nêÂ£o gasta um centavo.
- **Chamada real ao Jev** pelo OpenRouter: decisêÂ£o `reembolso`, confianêÂ§a 1,0, **610 ms**,
  **US$ 0,0000155**. A segunda chamada idêÂªntica veio do cache: **0 ms, US$ 0**.
  EvidêÂªncia: [`reports/smoke-2026-09-22.jsonl`](reports/smoke-2026-09-22.jsonl).
- Isso êÂ© teste de integraêÂ§êÂ£o e de contrato ê¢â¬â **nêÂ£o êÂ© benchmark de qualidade de decisêÂ£o**.

## O que este gateway nêÂ£o êÂ©

**NêÂ£o êÂ© proxy de agente de cêÂ³digo.** Ele nêÂ£o fica entre o seu Codex/Claude Code e o modelo.
Para isso existe o [jev-gateway](https://github.com/vinilana/jev-gateway) (projeto
independente, TypeScript), que êÂ© complementar a este.

**NêÂ£o executa aêÂ§êÂµes.** Devolve sugestêÂ£o; quem aplica êÂ© o seu sistema, com as permissêÂµes dele.

Lista completa em [ARQUITETURA.md, seêÂ§êÂ£o 6](ARQUITETURA.md#6-o-que-este-gateway-nêÂ£o-faz).

## Relacionados

- [Jev Decision Lab](https://github.com/linnoff88-dev/jev) ê¢â¬â o laboratêÂ³rio: 20 casos, 17 pacotes,
  experimentos e mêÂ©tricas. êâ° de lêÂ¡ que vem o cliente Jev usado aqui (vendorizado).
- [Curso Jev & Laya](https://linnoff88-dev.github.io/arckhe-/) ê¢â¬â decisêÂµes estruturadas na prêÂ¡tica.
- [Acervo Jev no Eventos ARCKHÉ](https://eventos.ARCKHÉ.pro/jev/).

## LicenêÂ§a

MIT. O cliente vendorizado (`jev_gw/vendor/jev_core.py`) vem do Jev Decision Lab, tambêÂ©m ARCKHÉ.
"jev" êÂ© o modelo da TypeSafe; este projeto êÂ© um cliente da API pêÂºblica dela, sem vêÂ­nculo.

---

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="32" />
  <p><strong>Arckhé AI</strong></p>
</div>