# FALHAS — jev-gw

| data | o que quebrou | menor correçêo | prompt \| infra |
|---|---|---|---|
| 2026-09-22 | push do portal, ARCKHÉbuscas e ARCKHÉpro-mono rejeitado (outra sessêo publicou no meio) — rebase deu conflito em arquivos GERADOS | resolver os gerados pelo lado de cima e REGENERAR (`gen:data`, `build.mjs`, `gen-catalog`), nunca editar o conflito à mêo | infra |
| 2026-09-22 | `git push origin` no ARCKHÉbuscas foi pro repo errado: ali `origin` = linnoff88-dev/ARCKHÉPRO; o upstream real é o remote `ARCKHÉbuscas` | conferir `git rev-parse --abbrev-ref main@{upstream}` antes de empurrar (liçêo já registrada no wifi/CLAUDE.md) | prompt |
| 2026-09-22 | teste de TTL do cache falhava por corrida (escrevia e lia no mesmo instante, com ttl=0.0001s) | envelhecer o arquivo com `os.utime` em vez de depender do relógio | prompt |
| 2026-09-22 | `core.py` vendorizado quebrou no import (`from . import __version__` nêo existe fora do pacote jev_lab) | o script de sincronizaçêo troca essa linha por uma constante | prompt |
