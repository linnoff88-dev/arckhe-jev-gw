# ðŸšª jev-gw â€” gateway de decisÃ£o para o Jev

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="48" />
  <p><strong>Arckhé AI</strong></p>
</div>


**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

[![jev-gw](guia/assets/banner.jpg)](https://inematds.github.io/jev-gw/guia/)

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/jev-gw/guia/**

Porta Ãºnica para toda consulta ao [Jev](https://docs.typesafe.ai/introduction) de um sistema:
valida o pedido, respeita um **teto de gasto diÃ¡rio**, aproveita **cache**, aplica a **polÃ­tica
de abstenÃ§Ã£o** e **registra custo e latÃªncia** de cada chamada.

E o principal: **quando algo falha, devolve revisÃ£o humana em vez de quebrar quem chamou.**
Jev fora do ar, chave errada, timeout, teto estourado â€” o seu sistema continua de pÃ©.

SÃ³ biblioteca padrÃ£o do Python 3.10+. **Nada para instalar.**

> **Como funciona por dentro: [ARQUITETURA.md](ARQUITETURA.md)** â€” o caminho de uma decisÃ£o,
> por que a ordem das etapas Ã© essa, o contrato de falha e o que o gateway nÃ£o faz.

## Incorporar no seu sistema

**Python â€” importa e usa.** Sem processo extra, sem porta.

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
                'reembolso': 'Pedido de devoluÃ§Ã£o ou estorno',
                'suporte': 'DÃºvida de uso ou defeito',
                'insuficiente': 'NÃ£o dÃ¡ para decidir com o que estÃ¡ escrito',
            },
        },
    },
})

if saida['acao'] == 'suggest':
    encaminhar(saida['resposta']['answers']['fila']['choice'])
else:
    fila_de_revisao(saida['motivo'])       # inclui "o Jev caiu" â€” e estÃ¡ tudo bem
```

**Qualquer outra linguagem â€” um POST.**

```bash
python3 -m jev_gw servir          # 127.0.0.1:8770
curl -X POST localhost:8770/decidir -H 'content-type: application/json' -d @pedido.json
```

**Terminal, cron, pipeline.** SaÃ­da `0` = sugestÃ£o, `2` = precisa de revisÃ£o.

```bash
python3 -m jev_gw decidir pedido.json
```

Exemplos rodando, um por forma: [`exemplos/`](exemplos/) (Python, `curl`, Node â€” todos sem
dependÃªncia).

## Instalar

```bash
git clone https://github.com/inematds/jev-gw.git
cd jev-gw
python3 -m unittest discover -s testes     # 23 testes, nenhum gasta crÃ©dito
```

Para usar de outro projeto, aponte o `PYTHONPATH` para a pasta, ou copie o diretÃ³rio
`jev_gw/` para dentro do seu projeto â€” Ã© autocontido de propÃ³sito.

A chave sÃ³ Ã© necessÃ¡ria na primeira consulta real:

```bash
export TYPESAFE_API_KEY=...        # ou OPENROUTER_API_KEY + JEV_PROVIDER=openrouter
```

## O que vocÃª ganha

| | |
|---|---|
| **Teto de gasto** | `JEV_GW_TETO_DIARIO=1.00`. Estourou, vira revisÃ£o â€” nÃ£o consulta mais |
| **Cache** | pedido idÃªntico em 15 min nÃ£o gasta de novo (medido: 610 ms â†’ 0 ms, US$ 0) |
| **Falha conservadora** | nenhuma falha de infraestrutura sobe como exceÃ§Ã£o para vocÃª |
| **Registro** | JSONL com custo, latÃªncia e decisÃ£o por chamada |
| **Painel** | `http://127.0.0.1:8770/painel` â€” gasto do dia, latÃªncia, Ãºltimas chamadas |
| **Privacidade** | o `state` (seus dados) **nÃ£o** vai para o registro. Escuta sÃ³ em `127.0.0.1` |

## Comandos

```bash
python3 -m jev_gw servir                  # sobe o serviÃ§o + painel
python3 -m jev_gw decidir pedido.json     # uma decisÃ£o
python3 -m jev_gw saude                   # estado e teto, sem gastar nada
python3 -m jev_gw custo --dia 2026-09-22  # resumo do dia
python3 -m jev_gw cache --limpar          # manutenÃ§Ã£o
```

## Testado de verdade (22/09/2026)

- **23 testes** passando, todos com avaliador controlado â€” a suÃ­te nÃ£o gasta um centavo.
- **Chamada real ao Jev** pelo OpenRouter: decisÃ£o `reembolso`, confianÃ§a 1,0, **610 ms**,
  **US$ 0,0000155**. A segunda chamada idÃªntica veio do cache: **0 ms, US$ 0**.
  EvidÃªncia: [`reports/smoke-2026-09-22.jsonl`](reports/smoke-2026-09-22.jsonl).
- Isso Ã© teste de integraÃ§Ã£o e de contrato â€” **nÃ£o Ã© benchmark de qualidade de decisÃ£o**.

## O que este gateway nÃ£o Ã©

**NÃ£o Ã© proxy de agente de cÃ³digo.** Ele nÃ£o fica entre o seu Codex/Claude Code e o modelo.
Para isso existe o [jev-gateway](https://github.com/vinilana/jev-gateway) (projeto
independente, TypeScript), que Ã© complementar a este.

**NÃ£o executa aÃ§Ãµes.** Devolve sugestÃ£o; quem aplica Ã© o seu sistema, com as permissÃµes dele.

Lista completa em [ARQUITETURA.md, seÃ§Ã£o 6](ARQUITETURA.md#6-o-que-este-gateway-nÃ£o-faz).

## Relacionados

- [Jev Decision Lab](https://github.com/inematds/jev) â€” o laboratÃ³rio: 20 casos, 17 pacotes,
  experimentos e mÃ©tricas. Ã‰ de lÃ¡ que vem o cliente Jev usado aqui (vendorizado).
- [Curso Jev & Laya](https://inematds.github.io/jev-curso/) â€” decisÃµes estruturadas na prÃ¡tica.
- [Acervo Jev no Eventos INEMA](https://eventos.inema.pro/jev/).

## LicenÃ§a

MIT. O cliente vendorizado (`jev_gw/vendor/jev_core.py`) vem do Jev Decision Lab, tambÃ©m INEMA.
"jev" Ã© o modelo da TypeSafe; este projeto Ã© um cliente da API pÃºblica dela, sem vÃ­nculo.

---

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="32" />
  <p><strong>Arckhé AI</strong></p>
</div>