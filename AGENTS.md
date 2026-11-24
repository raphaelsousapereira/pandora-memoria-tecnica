# Regra de trabalho PANDORA

Antes de planejar, editar, diagnosticar ou publicar, abra MEMORIA.md na revisão atual da main. Leia as entradas sobre o sistema afetado e as falhas que tenham sintomas semelhantes. No início do registro de trabalho, cite a revisão consultada, os IDs aplicáveis e o primeiro teste que distingue as causas. Se nenhum caso corresponder, registre a busca e prossiga. Se não tiver acesso, indique a limitação e obtenha a cópia autorizada antes de afirmar que consultou o histórico.

Ao encerrar, compare o resultado com a condição de aceite do usuário. Se houve descoberta, tentativa que falhou, correção ou nova evidência, acrescente uma entrada usando MODELO.md. Para um caso existente, acrescente uma retificação datada que cite seu ID. Preserve a sequência histórica. Inclua na entrega o link do commit da memória ou identifique a atualização pendente. Para trabalho rotineiro sem descoberta, registre no PR a justificativa curta. Revise comandos históricos antes de aplicá-los a outro serviço. Proteja credenciais, identificadores de clientes, áudio e transcrições privadas.

<!-- pandora-governanca:inicio -->
## Governança do desenvolvimento (PANDORA GOVERNANÇA)

Toda mudança segue **Declarar → Estruturar → Gerar → Rastrear**. Guia: `docs/GOVERNANCA.md` em
raphaelsousapereira/pandora-saas-forge.

1. **Declarar.** Antes de codar, escreva os itens em `specs/` (`SPEC-<ÁREA>-<n>`, cada um com critério de aceite).
   Mudança trivial: uma linha `Sem spec: <motivo>` no texto do PR. Item de outro repositório: `SPEC-GOV-1@pandora-saas-forge`.
2. **Estruturar.** Dependências permitidas, caminhos proibidos e arquivos gerados estão em `.pandora/arquitetura.json`.
   Dependência nova entra no contrato no mesmo PR, com o motivo no texto do PR.
3. **Gerar.** Escreva o texto do PR em um arquivo e rode
   `python tools/pandora_governanca.py verificar --pr-corpo <arquivo>` até sair `APROVADO`.
4. **Rastrear.** Último commit do PR: `python tools/pandora_governanca.py registrar --pr-corpo <arquivo>`. Ele grava o
   BOM de proveniência em `governanca/bom/` e acrescenta uma linha encadeada por hash em `governanca/registro.jsonl`.
   Nunca edite linhas antigas. Conflito no registro: atualize o ramo com o principal, descarte a sua linha e rode `registrar` de novo.

O texto do PR sempre traz: itens `SPEC-…` atendidos, `Memória: <commit do MEMORIA.md consultado>`, testes executados e
o resultado do portão. A aprovação humana é o review e o merge do PR: nada vai direto para o ramo principal.
Não edite `tools/pandora_governanca.py` aqui; ele é instalado a partir do pandora-saas-forge.
<!-- pandora-governanca:fim -->
