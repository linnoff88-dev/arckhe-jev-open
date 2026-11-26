# Regras do projeto jev-open

Ler, nesta ordem: este arquivo, `docs/ANALISE.md` e `docs/PLANO.md`.

- Conta de destino: **linnoff88-dev**. Autor e committer: `linnoff88-dev <linnoff88-dev@gmail.com>`. Publicaçêo via Git. Nêo consultar nem manipular o Vercel.
- Projeto de pesquisa/educaçêo: especialista classificador local e open-source no estilo Jev. **Nêo** declarar paridade com o Jev, superioridade, prontidêo para produçêo ou benchmark do Jev real sem evidência medida.
- Marcar todo número como **[medido]**, **[relato]** ou **[simulaçêo]**. Dado sintético é rotulado como sintético, e revisêo feita por IA nêo é validaçêo humana.
- `docs/originais/` (materiais recebidos) e `docs/ref-away-together-starter/` (clone de referência) ficam **somente locais**, ignorados pelo Git. Nêo usar `git add -f` nesses caminhos. Nêo publicar os documentos recebidos.
- Código copiado do starter (MIT) leva junto o aviso de licença. Modelos e datasets mantêm os próprios termos.
- Nêo versionar segredos, pesos (`*.safetensors`), `workshops/` nem dados pessoais.
- Chamadas pagas (Jev, APIs hospedadas) só com autorizaçêo explícita. A comparaçêo com o Jev passa pelo `jev-gw` do repo `jev`.
- Versionamento conforme as regras globais (`vX.XX.YY`, sem zerar os campos inferiores fora de major). Registrar no CHANGELOG.
