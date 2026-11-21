# Memória técnica acumulativa da PANDORA

Índice adicional: ENCONTROS-2026-001 · reuniões · gravação longa · reenvio · decisões · fatos editoriais.

Índice pesquisável: AUDIO-2026-001 · Dora · microfone oculto · Render live · Whisper · feature flag · Chromium · transcrição em português. Acrescente os próximos projetos ao índice e ao final do arquivo.

## 2026-11-20 · AUDIO-2026-001 · O portal estava online e o microfone continuava oculto

### Pedido e aceite

Raphael queria gravar no campo da Dora, transcrever com um motor Python reutilizável e revisar antes de enviar. O aceite inclui abrir a tela autenticada, iniciar a gravação, interrompê-la, receber o texto para revisão e decidir quando enviar. Cancelar deve descartar a gravação. Há três componentes independentes: captura no navegador, endpoint autenticado do portal e inferência local pelo Whisper. O sucesso de um componente não comprova o seguinte.

### O que os testes realmente cobriram

1. O motor foi exercitado primeiro em isolamento. Testes com componente simulado permitiram verificar limites e falhas previsíveis. Esses testes não medem reconhecimento de fala.
2. Um WAV público de 17,76 segundos convertido em WebM/Opus percorreu a rota HTTP com modelo local real em 6,839 segundos. O cenário retornou 200 para áudio válido, 401 sem autenticação e 403 para outro tenant. As identidades eram fixtures. Este valor é uma observação individual e não uma estimativa de P95 ou SLA.
3. No Render, o teste de navegador executou Chromium com microfone virtual e modelo real. O contador registrou uma requisição de áudio e uma conversa após o envio explícito. O cancelamento também foi exercitado. O servidor desse teste usava sessão de teste. A instalação temporária do modelo naquele processo não garantia sua presença depois de um deploy.
4. Na validação anterior ao merge, 33 testes direcionados passaram. A comparação da suíte Python encontrou 22 falhas comuns na base e na alteração testadas. Sete verificações visuais locais ficaram prejudicadas quando o Chromium não conseguiu criar socket no ambiente restrito. Ao documentar isso, identificar o ambiente e o ponto de falha. O teste específico de áudio em Chromium no Render não substitui as outras suítes visuais.
5. O PR #245 ainda estava em rascunho enquanto produção servia o commit 21481a9 do PR #246 (matriz de planos). O número maior do outro PR não indicava inclusão do áudio. Conferir a revisão entregue e o estado merged.

### As três publicações e por que a primeira resposta foi prematura

Horários de Brasília em 20/11/2026. O commit de merge em main foi 903bfb9502fdbfc6a4dcd732646f003940704f3e.

| Hora | Evidência | Diagnóstico que passou a ser possível |
|---|---|---|
| 17h52 | Deploy dep-dc0b5gd9el5s73fr29sg ficou live | Código do PR chegou ao serviço. O health check do portal podia passar sem validar a transcrição. |
| 18h04 | Deploy dep-dc0b9rd7phjs73agu0g0 ficou live | O serviço passou a instalar Whisper e provisionar o modelo. A variável do caminho foi gravada. Ainda faltava conferir a chave de ativação. |
| 18h10 | Deploy dep-dc0bdpon74is73b2e62g ficou live; diagnóstico do build registrou ready_for_load=true | O servidor recebeu a configuração de ativação. O diagnóstico indicou que os arquivos e dependências estavam prontos para carregar. |
| Após o deploy | Raphael confirmou o funcionamento e mostrou o ícone e a mensagem “Transcrição pronta. Revise o texto e envie quando quiser.” | Confirmação de usuário no produto. A imagem mostra o estado final da interface. Não contém a frase falada e não serve para medir acurácia. |

### Causa e correção reproduzível

O build efetivamente cadastrado no painel apenas instalava requirements.txt, requirements-navegador.txt e Chromium. O render.yaml da main já descrevia um comando mais completo. Conferir sempre os dois valores. O comando aplicado ao serviço foi:

```sh
pip install -r requirements.txt -r requirements-navegador.txt '.[whisper]' &&
playwright install --only-shell chromium &&
mkdir -p .audio-models &&
python -m pandora_audio.provision --destination .audio-models/small --download &&
python -m pandora_audio.doctor --model .audio-models/small
```

No serviço web pandora-cloud, configurar as duas variáveis abaixo. Fazer atualização pontual para preservar as demais variáveis. Validar o caminho no serviço que efetivamente executa o portal.

```dotenv
PANDORA_AUDIO_MODEL_PATH=/opt/render/project/src/.audio-models/small
PANDORA_AUDIO_ENABLED=1
```

A função available() de pandora_cloud/assistant_audio.py retorna falso se a chave de ativação for diferente da string 1. Depois consulta ready_for_load para o caminho configurado. A rota de status fornece audio_enabled. O JavaScript ainda exige navigator.mediaDevices, getUserMedia e MediaRecorder para revelar o botão. Assim, instalar o modelo e configurar apenas seu caminho deixou a interface sem microfone. O último ajuste foi ativar a chave, acompanhar o novo deploy e reabrir a interface. O trabalho terminou com o relato do usuário.

### Sequência para a próxima ocorrência

1. Anotar URL do produto, serviço, branch, SHA e hora. Comparar o SHA esperado com o deploy live. Se divergirem, localizar o PR, seu merge e o deploy correspondente antes de mexer no modelo.
2. Com SHA correto, abrir o status do assistente na própria sessão autenticada. Consultar somente audio_enabled. Um redirecionamento para login indica ausência de sessão para esse teste. Preservar a autenticação e usar uma conta de teste autorizada.
3. Com audio_enabled falso, conferir as duas variáveis e o comando ativo do build. No log do deploy correto, localizar saída de pandora_audio.doctor, presença de model.bin, config.json e tokenizer.json e valor de ready_for_load. Se o diagnóstico falhar, corrigir dependências ou caminho e refazer o build. Um log de deploy antigo não demonstra o estado do atual.
4. Com audio_enabled verdadeiro e botão oculto, recarregar a tela e conferir no navegador HTTPS, mediaDevices, getUserMedia e MediaRecorder. Se aparece mas não grava, verificar permissão de microfone do site, dispositivo selecionado e erro apresentado. Alterar o backend apenas se houver evidência de falha nele.
5. Com gravação concluída, verificar no painel de rede o POST /api/assistente/audio/transcrever e status. Tratar autenticação, autorização, limites ou erro de processamento conforme a resposta concreta. Evitar guardar cookies ou conteúdo do áudio em tickets.
6. Testar uma frase curta conhecida. Verificar texto no campo, ausência de envio automático, cancelamento, envio manual único. Registrar navegador, ambiente, revisão e resultado. Se a verificação usou fixture, escrever isso. Se o usuário validou em produção, distinguir sua confirmação da execução automatizada.
7. Se for necessário conter uma falha de voz, desativar somente PANDORA_AUDIO_ENABLED e verificar o funcionamento do chat de texto. Se houver regressão geral, identificar o último deploy saudável e avaliar também as variáveis persistentes antes do rollback. Não houve rollback observado neste caso.

### Decisões para poupar tentativas

Verificar status e variáveis antes de propor mais RAM, disco ou plano de workspace. Os 25 GB da tela de planos se referiam a bandwidth. O serviço consultado usava 1 CPU e 2 GB de RAM. Esses números descrevem o caso e não dimensionam outros produtos. Erro “Load failed (api.render.com)” no painel deve ser investigado separadamente do domínio do SaaS; conferir estado do serviço por outro acesso autorizado e comparar horário e deploy. Sem cota de GitHub Actions, usar a validação local indicada pelo repositório, registrar falhas por suíte e procurar um ambiente capaz de executar o teste específico que ficou bloqueado.

### Evidências primárias

- [PR #245 e arquivos da implementação](https://github.com/raphaelsousapereira/pandora-cloud/pull/245)
- [Código publicado](https://github.com/raphaelsousapereira/pandora-cloud/tree/903bfb9502fdbfc6a4dcd732646f003940704f3e)
- [Deploy de ativação](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0bdpon74is73b2e62g)
- Captura fornecida pelo usuário, “Captura de tela 2026-11-20 181254.png”, e confirmação nesta conversa. O original ficou na conversa.

## 2026-11-20 · ENCONTROS-2026-001 · Captura de reuniões por etapa e separação editorial

**Origem.** Projeto pandora-cloud. O caso AUDIO-2026-001, consultado na versão ff2a5deb da memória, comprovava ditado curto com revisão. O novo pedido exige persistência por cliente e etapa, consulta posterior, decisões confirmadas e seleção de fatos para uso no site. Primeira verificação escolhida: salvar uma parte WebM, relê-la pelo endpoint autenticado e enviá-la ao motor local real.

**Implementação e pontos de entrada.** Em Assistente ou Visão geral, selecionar o site e abrir “Encontros do projeto”. O formulário aceita projeto, etapa, participantes e o registro da concordância. A rota /encontros usa a sessão do portal. A API /api/encontros verifica o site e limita cada encontro ao usuário criador, organização e site. Na interface, projeto e etapa são campos do encontro; não representam uma sincronização com uma entidade de CRM ou contrato. O histórico ordena os registros pela data de criação.

**Como tratar mídia.** Capturar partes independentes com MediaRecorder e preservar o contêiner completo de cada gravação. No navegador, salvar o blob na IndexedDB antes da transferência. A chave de reenvio é encontro + sequência. O servidor compara SHA-256, aceita repetição do conteúdo e rejeita conteúdo diferente na mesma posição com 409. Registrar a posição de início e a duração medida. Evitar multiplicar a duração nominal pelo número de eventos. A entrega de eventos pode atrasar devido ao navegador ou à suspensão do dispositivo. A troca entre gravações pode introduzir pequenas lacunas; qualificar isso com dispositivos reais antes de prometer continuidade acústica.

**Sincronização, privacidade e capacidade.** Após pausar, o cliente solicita a transcrição das partes por ordem. O botão de sincronização retoma trechos pendentes quando a mesma conta reabre o mesmo encontro no mesmo navegador. Fechar a página interrompe a captura e o processamento comandado pelo cliente. A parte em gravação ainda pode estar em memória. Usar a sequência e o estado de cada trecho para informar o último progresso conhecido. O modelo real e o controle de concorrência são os mesmos do ditado. O limite inicial de dados de áudio é 128 MiB agregados na base do serviço. SQL NULL libera os blobs para reúso interno do SQLite; isso não reduz imediatamente o tamanho do arquivo físico nem elimina backups. O comando de exclusão do áudio exige que todos os trechos já estejam transcritos. Para crescer: dimensionar retenção e concorrência por cliente, avaliar armazenamento de objetos e worker com fila persistente. Não alterar planos com base apenas no rótulo da hospedagem.

**Decisão e origem.** Guardar transcrição, correção, marcação, decisão confirmada e fato editorial como registros distintos. A correção preserva o texto original. A consulta de encontros da Dora recupera trechos privados no escopo do usuário com referência de projeto, data, etapa e posição. O fluxo editorial copia apenas os fatos marcados para SEO/GEO. Não enviar automaticamente conversa privada a um provedor ou ao site do cliente. Registrar quem revisou e qual parte originou a decisão. Nesta versão a identidade é a do proprietário do encontro; não há identificação automática dos interlocutores.

**Testes realizados.** FastAPI TestClient, base SQLite temporária e identidades de teste. Um WAV foi convertido em WebM/Opus, salvo, lido byte a byte e transcrito pelo modelo local real. Verificados 401 sem identidade, 403 para site indevido, 404 para outro usuário do mesmo site, rejeição de criação sem concordância, reenvio idêntico, conflito na mesma sequência, bloqueio da exclusão com transcrição pendente, idempotência das notas e recuperação de uma decisão pelo texto. A linha do tempo aceitou 180 partes de material repetido com posições até 59:40. Essa simulação exercita armazenamento e ordenação. Não comprova gravação física de uma hora, perda real de rede, qualidade do microfone no Safari, duração da bateria ou ausência de lacunas. Na validação de campo, testar desconexão/retorno, pausa, bloqueio de tela, permissão revogada, armazenamento local cheio e recarga com partes pendentes.

**Publicação e investigação de recorrência.** Conferir SHA de merge no serviço Render. Um serviço live atesta o deploy. Só confirmar a jornada autenticada depois de gravar, reabrir o histórico, reproduzir um trecho, consultar sua origem e verificar o material editorial com uma conta autorizada. Se o botão de ditado funcionar e o de encontros não aparecer, conferir HTML e JS do mesmo deploy, o site escolhido e o escopo de sessão. Não aumentar a franquia do ditado curto para tentar convertê-lo em gravador de reunião.

- [PR #249 e código](https://github.com/raphaelsousapereira/pandora-cloud/pull/249)
- [Deploy do commit fa4cc24](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0bp2btqb8s73daopag)
- [Referência do navegador para dataavailable e atrasos de timeslice](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder/dataavailable_event)

### 2026-11-20 · ENCONTROS-2026-002 · Administração por cliente, inclusive sem site

Na área Clientes e sites, abrir Encontros do cliente na linha da organização. A tela /admin/encontros?org=… valida platform_admin e a existência da organização, exibe o cliente e preenche o projeto. Reutiliza a captura, fila, transcrição e persistência do caso 001. Para consulta posterior, retornar pelo mesmo cliente com o mesmo administrador. A organização e o usuário formam o escopo dos registros administrativos, com site vazio. Não inventar site fictício para uma reunião de contrato, não inferir autorização por ID na URL e não combinar automaticamente os registros privados de usuários diferentes.

Em SQLite temporária com identidades de teste: cliente sem site abriu a ficha; áudio WebM foi persistido e reproduzido byte a byte; Whisper real transcreveu e a consulta posterior recuperou o texto. Um usuário comum recebeu 403 e a consulta no escopo de site não encontrou o encontro administrativo. O teste não é uma sessão autenticada no portal publicado. [PR #250](https://github.com/raphaelsousapereira/pandora-cloud/pull/250).

## 2026-11-20 · VOZ-2026-001 · Ouvir a resposta da Dora

O microfone e Whisper atendem entrada de voz. Para reproduzir uma resposta já escrita, o frontend conecta o texto ao módulo ES pandora-voz.mjs. Disponibilizar o módulo com tipo JavaScript. attachReader cria controle por resposta; stopReading interrompe ao trocar cliente, iniciar microfone ou abrir nova conversa. A lista de vozes é consultada no clique. Usar voz local compatível, manter o texto legível e informar como habilitar uma voz se o dispositivo não a tiver. Não presumir que síntese do navegador e voz instalada são equivalentes. Para reutilizar em outro projeto, seguir docs/PANDORA_VOZ.md e testar com som no navegador alvo.

Teste Node com sintetizador simulado verificou integridade de texto longo dividido em partes, cancelamento sem retomar por callback antigo e ausência de envio a voz remota. Teste simulado não comprova qualidade acústica nem disponibilidade de voz no dispositivo do cliente. [PR #251](https://github.com/raphaelsousapereira/pandora-cloud/pull/251).

## 2026-11-20 · APROVACOES-2026-001 · Dois portais, aceite da versão exata

PANDORA Cloud trabalha com organização, site e papéis operator/client_admin. Arquitetura usa projeto e project_members; a equipe elabora e o cliente decide. A listagem por proprietário revelou arquitetura-clientes-saas. Evitar deduzir ausência de um produto a partir de busca parcial pelo nome. O painel de arquitetura já tinha decisões simples; mantivemos esse fluxo e acrescentamos as fichas versionadas em Decisões. No Cloud o acesso parte de Aprovações na Dora com o site selecionado.

Componente JavaScript compartilhado por adaptador, com edição/revisão e resposta independentes. A decisão do cliente está vinculada ao item e ao número da versão. Registrar ator e hora no servidor, bloquear resposta duplicada e resposta para versão antiga. Preservar versões anteriores. Usar uma ficha por foto ou texto se forem aprováveis individualmente. Incluir na ficha apenas a referência da reunião autorizada; links de arquivos devem apontar a versões estáveis. A decisão de adiar mantém a pendência.

Erro evitado: schema.sql histórico citava public.project_access; a inspeção das políticas efetivas mostrou private.project_access. A migração aditiva foi aplicada no Supabase do portal de arquitetura e registrada no repositório. A gravação PostgreSQL usa lock por projeto/item; SQLite usa a transação de escrita do workspace. Teste em Supabase dentro de transação revertida criou identidades descartáveis e simulou duas aprovações seguidas de revisão de uma única ficha. No SQLite verificamos a nova pendência, a preservação do outro aceite, recusa por papel, conflito de versão, isolamento e recuperação pela Dora. JS passou na verificação sintática; a jornada no celular com uma conta real de cada papel ainda precisa de validação. Não confundir deploy live com aceite de usuário. Para reutilização e investigação consultar docs/CENTRAL_APROVACOES.md em cada projeto.

- [PANDORA Cloud PR 252](https://github.com/raphaelsousapereira/pandora-cloud/pull/252)
- [Arquitetura PR 4](https://github.com/raphaelsousapereira/arquitetura-clientes-saas/pull/4)

### ENCONTROS-2026-003 · Exportação e exclusão seletiva

Ações em Áudio e transcrição: impressão do texto, cópia e TXT UTF-8 do encontro selecionado. Incluir horários, pendências e correções identificadas, separar do botão de copiar fatos editoriais. Ao trocar de encontro, invalidar a seleção carregada antes de exportar. Excluir só o blob solicitado, vinculando proprietário, encontro e sequência; preservar texto e notas. Validado em SQLite com duas reuniões e trecho pendente, usando dados de teste. Diálogo de impressão, clipboard e download em iPhone requerem verificação no dispositivo. [PR #259](https://github.com/raphaelsousapereira/pandora-cloud/pull/259).

## VIDEO-2026-001 · Studio de vídeo por instrução em português

O pacote pandora_video usa FFmpeg e Whisper local. pandora-motion contém efeitos de interface, portanto um efeito CSS não aparece automaticamente no MP4 exportado. Aplicar os filtros ao fluxo de vídeo, verificar as trilhas finais e medir a duração com ffprobe. Formatos, duração, tempo de execução e concorrência devem ser limitados. Usar argumentos separados no subprocesso e arquivos temporários privados. Separar JS/CSS do HTML para compatibilidade com CSP. Um teste local com gravação de fala e imagem gerada produziu MP4 H.264/AAC de cinco segundos com corte, filtro de legendas e fade. O teste da rota com identidade de teste retornou video/mp4. Validar separadamente uma sessão real no celular, orçamento de CPU/RAM com carga simultânea e prontidão do modelo em cada SaaS. [PR #260](https://github.com/raphaelsousapereira/pandora-cloud/pull/260).

### VIDEO-2026-002 · Integração com clientes MCP

Encontrado servidor remoto OAuth preexistente em /mcp. O registro da ferramenta preparar_edicao_video permite aos clientes montar o pedido, validar duração e abrir o editor com a instrução preenchida. Reutilizar o fluxo de consentimento existente. Não presumir que um endpoint publicado implica instalação numa conta ou acesso aos arquivos anexados no chat. Renderização web exige selecionar o arquivo e confirmar no Studio. Testados o link e a validação de argumentos; autenticação real dentro de cada cliente é um aceite separado. [PR #261](https://github.com/raphaelsousapereira/pandora-cloud/pull/261).

## Studio de vídeo · estilos de legenda · 20/11/2026
Manter o catálogo no pacote Python e fornecer ao navegador os mesmos IDs usados na exportação. A prévia usa CSS e pode variar conforme a fonte do aparelho. O renderizador só aceita IDs cadastrados e converte as cores RGB para a ordem BGR do libass. Preservar o comportamento de pedidos sem estilo. Validar quantidade e unicidade dos estilos e exportar um vídeo com fala real. No PR #262 o teste local com Tarja · Oceano gerou seis segundos de H.264/AAC. A integração fica em pandora_video e requer o modelo de transcrição disponível no serviço.


## 2026-11-20 · ORQUESTRADOR-2026-001 · Aprendizagem supervisionada entre agentes e publicação de código completo

- **Projeto, componente, ambiente e revisão:** \`pandora-corpo/packages/pandora_orchestrator\`, v0.4.2; SQLite local; merge \`b5ed5c44153e0b7424fe7fb36c12f57fb0816acd\`, PR #159. Data no fuso America/Sao_Paulo.
- **Pedido e aceite:** publicar no GitHub toda evolução do orquestrador v0.4.1, permitindo que um agente pesquisador acesse páginas públicas autorizadas, extraia evidências e as compartilhe com outros agentes sem autonomia para promover instruções externas.
- **Memória consultada antes da mudança:** revisão \`8e7707d7fb819c42f41c3adef04b03b9f16d13d5\` (esta memória); nenhum caso anterior específico de Knowledge Network identificado. Teste inicial: 76 regressões herdadas e cenários de coleta/aprovação/ensino.
- **Implementação:** pacote independente com 36 arquivos no \`pandora-corpo\`; \`knowledge_web.py\` faz leitura HTTPS pública opt-in, revalida IP público na conexão, verifica TLS/robots e bloqueia redirects; \`knowledge_network.py\` registra evidência bounded com hash/proveniência e status \`pending\`, aprova por hash, concede a agentes do mesmo org/site, recupera lexicalmente com fonte e TTL, revoga. \`api.py\` expõe cinco endpoints \`/v1/knowledge/*\`. A coleta é desligada por padrão (\`ORCH_KNOWLEDGE_WEB_ENABLED\`).
- **Riscos controlados:** conteúdo remoto sempre é evidência não confiável, nunca comando nem ajuste de pesos; sem compartilhamento entre tenants, sem publicação automática, sem execução de instruções da web. DNS privados/mistos, IPs literais, URLs não HTTPS e dados acima de 512 KiB são rejeitados. Aperfeiçoamento de última hora: confirmar integridade de URL+trecho no aceite e na recuperação, não apenas confiar no hash armazenado.
- **Evidência e testes:** \`cd packages/pandora_orchestrator && python -m pytest -q\` → 94 passed; \`python -m compileall -q orchestrator\` → passou. Testes incluem robots, SSRF, hashes alterados, isolamento, autorização, deduplicação, expiração e revogação. HTTP e conectores externos foram simulados. Nenhum GitHub Actions nem API de IA paga foi empregado.
- **Limitações:** não há prova de execução com sites reais, browser JavaScript, credenciais OAuth delegadas, PostgreSQL remoto ou SaaS em produção; a presença do código em GitHub não significa deploy. Não converter essa implantação documental em aprovação para crawling multicliente.
- **Próxima ocorrência:** verificar estado de flags e tenant/site, executar suíte local, coletar uma página de teste explicitamente autorizada em staging, validar logs de robots/TLS e fazer testes de integração PostgreSQL + delegação OAuth antes de ativar produção.
- **Evidências:** [PR #159](https://github.com/raphaelsousapereira/pandora-corpo/pull/159), [código publicado](https://github.com/raphaelsousapereira/pandora-corpo/tree/b5ed5c44153e0b7424fe7fb36c12f57fb0816acd/packages/pandora_orchestrator).


## 2026-11-20 · ORQUESTRADOR-2026-002 · Integração de homologação entre CLOUD e Orchestrator v0.4.2

- **Projeto e ambiente:** \`pandora-cloud\`, ponte HTTP de staging para \`pandora-corpo/packages/pandora_orchestrator\`. Origem do motor: commit \`b5ed5c44153e0b7424fe7fb36c12f57fb0816acd\`, PR #159. Integração Cloud: PR #263, merge \`6a0acb16b14f3db157afb97592fa33971ae91257\`. Fuso America/Sao_Paulo.
- **Pedido e aceite:** seguir da biblioteca de agentes para integração, preservando segregação cliente/site, sem liberar coleta ou execução autônoma e com todos os avanços no GitHub.
- **Memória consultada:** revisão \`e807502e7d440a976210fa3ccc9a1b630cf810d5\`, caso \`ORQUESTRADOR-2026-001\`. Primeiro teste: cliente/tenant incorreto não alcança backend, mesmo com credencial configurada.
- **Implementação:** \`pandora_cloud/orchestrator_bridge.py\` valida endpoint de staging HTTPS \`*.onrender.com\` fixo por configuração, token separado por org/site, respostas limitadas a 128 KiB e bloqueio de redirects/proxies. \`orchestrator_bridge_routes.py\` registra somente operações de observação: status, prévia determinística do plano e conhecimento previamente aprovado. Portal usa autenticação JWT, \`_require_admin\`, \`_belongs\` e \`audit_block\` antes de qualquer chamada remota. Nunca encaminha JWT do usuário.
- **Configuração de segurança:** \`PANDORA_ORCH_BRIDGE_ENABLED\` ausente ou diferente de \`1\` significa desativado. Habilitação depende de \`PANDORA_ORCH_STAGING_URL\` e mapa secreto \`PANDORA_ORCH_SITE_TOKENS_JSON\`. **Não** é OAuth delegado; proibido tratar essa ponte como liberação de coding agents, web ingest, aprovações ou MCP global em produção.
- **Testes locais:** 24 testes direcionados com \`httpx.MockTransport\` passaram, incluindo verificação de papel admin, organização/site, revogação, token ausente, URL não permitida, redirects, erros/oversize, modelo de tarefas estrito e ausência de operações mutáveis. Compilação dos dois módulos passou. **Suíte completa do Cloud, serviço remoto real e PostgreSQL não foram testados**.
- **Limites e aceite:** merge do código e deploy do portal não significam operação ponta a ponta do Orchestrator. Precisa haver uma instância staging com PostgreSQL persistente, credenciais escopadas e OAuth delegado para operações MCP antes de ativar. Comparar commit esperado com deploy Render \`dep-dc0eahdg1s2s73e06osg\` e confirmar status health em logs. Não gravar tokens em memória, GitHub ou screenshots.
- **Evidências:** [PR #263](https://github.com/raphaelsousapereira/pandora-cloud/pull/263), [documentação de configuração](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/PANDORA_ORCHESTRATOR_STAGING_BRIDGE.md).


## 2026-11-20 · ORQUESTRADOR-2026-003 · Staging Render isolado e plano de PostgreSQL

- **Projeto, componente, ambiente, revisão:** PANDORA ORCHESTRATOR Ω v0.4.2 em \`pandora-corpo\`; origem \`b5ed5c44153e0b7424fe7fb36c12f57fb0816acd\`. Staging público em https://pandora-orchestrator-staging.onrender.com, serviço Render \`srv-dc0ed6t9el5s73dro6og\`; deploy inicial \`dep-dc0ed759el5s73dro780\` marcado live. Banco Postgres 16 \`pandora-orchestrator-staging-db\`, Render \`dpg-dc0ecth7lnhs738cpgm0-a\`, gratuito e disponível; validade indicada por Render até 21/12/2026 UTC. Fuso America/Sao_Paulo.
- **Pedido e aceite:** avançar a integração do Orchestrator ao CLOUD sem tocar dados reais, garantir versionamento no GitHub, configurar homologação com segurança.
- **Memória consultada:** revisão \`3fe577b2d62bae91afeb29a5cda9fd5ee594c7b5\` e casos \`ORQUESTRADOR-2026-001\`/\`002\`; primeiro teste: verificar status do deploy independente e logs de inicialização; não confundir build live com health validado por cliente.
- **Execução real:** serviço gratuito criado a partir do GitHub, build editable Python 0.4.2 aprovado e Uvicorn iniciou; deploy initial live. Flags \`ORCH_MCP_READ_ENABLED=0\`, \`ORCH_KNOWLEDGE_WEB_ENABLED=0\`, \`ORCH_CODE_ENABLED=0\`. Staging usa \`ORCH_ENV=development\` e SQLite descartável em /tmp; nenhuma credencial ou conteúdo real de cliente deve ser armazenado ali. Foi criado token sintético forte com hash SHA256, papel admin escopado a IDs de teste sem vínculo com clientes reais; não copiar token em memória ou logs.
- **Banco e limitação de conector:** Postgres isolado foi criado e ficou disponível, mas a integração Render MCP não expõe connectionString nem permite \`fromDatabase\` em \`update_environment_variables\`. Consulta direta à base por Render MCP foi negada por IP allowlist vazio (comportamento seguro). **Banco ainda NÃO vinculado ao serviço.**
- **Solução preparada:** PR [#160](https://github.com/raphaelsousapereira/pandora-corpo/pull/160) merge \`ab6506184c0513233e376a7d95dab7a121d26571\`, adiciona \`deploy/orchestrator-staging.render.yaml\` e documentação. Render Blueprint \`exs-dc0een6k1f9s738ssu40\`, \`existingResources=adopt\`, \`autoSync=false\`: planejamento \`planned\` com referência segura \`ORCH_DATABASE_URL.fromDatabase\` ao banco existente e mudança para \`ORCH_ENV=production\`. **Nenhuma aprovação de sincronização foi executada**, preservando serviço atual. Plano: https://dashboard.render.com/blueprint/exs-dc0een6k1f9s738ssu40/sync/exe-dc0eenek1f9s738ssumg. Aprovação exige revisão explícita do plano para não alterar serviços indevidamente.
- **CLOUD:** variáveis de ambiente da bridge atualizadas para \`PANDORA_ORCH_BRIDGE_ENABLED=1\`, URL staging fixa HTTPS e mapa \`PANDORA_ORCH_SITE_TOKENS_JSON={}\`. Isso permite somente consultar health/status pela rota administrativa autenticada; nenhum tenant tem credencial, e plano/knowledge ficam bloqueados (409). A atualização disparou deploy \`dep-dc0ee5mk1f9s738sr3n0\`; status final deve ser verificado separadamente. Não habilitar crawling, coding agents nem MCP global.
- **Limites de verificação:** primeiro deploy do serviço live e logs Uvicorn, porém o endpoint /health não pôde ser verificado externamente pelo navegador de pesquisa. Não houve smoke autenticado com Postgres nem dois tenants; não afirmar homologação ponta a ponta.
- **Próximos testes:** revisar/autorizar Blueprint, aplicar com fromDatabase sem segredos; confirmar produção/PG, reinício + persistência, credenciais escopadas por org/site e expiração; então testar ponte CLOUD com fixture e tenant real autorizado. Revisar deploy após atualização de env e logs.
- **Fontes:** Render docs https://render.com/docs/blueprint-spec; [runbook staging](https://github.com/raphaelsousapereira/pandora-corpo/blob/main/deploy/ORCHESTRATOR_STAGING_README.md); [PR #263 Cloud](https://github.com/raphaelsousapereira/pandora-cloud/pull/263).


### 2026-11-20 · Retificação de ORQUESTRADOR-2026-003 — dois deploys live confirmados

Os dois deploys que estavam pendentes atingiram \`live\` no Render, com logs de inicialização Python. Pandora Cloud: \`dep-dc0ee5mk1f9s738sr3n0\`, commit \`6a0acb16b14f3db157afb97592fa33971ae91257\`, \`live\` em 2026-11-21 00:36:17 UTC; dois GET /health 200 nos logs após o processo iniciar. Orchestrator staging: \`dep-dc0eelfb9u1s73bft03g\`, commit \`ab6506184c0513233e376a7d95dab7a121d26571\`, \`live\` em 00:36:24 UTC. Isso confirma implantação dos serviços, **não** conexão PostgreSQL, nem autenticação de um tenant pela ponte, nem testes autenticados ponta a ponta. Render Blueprint \`exs-dc0een6k1f9s738ssu40\` continua apenas \`planned\`, não aprovado. A persistência atual do staging permanece SQLite descartável.


### 2026-11-20 · Retificação ORQUESTRADOR-2026-003 — associação PostgreSQL aplicada

O pedido de continuar a integração foi seguido com a aprovação do Blueprint Render \`exs-dc0een6k1f9s738ssu40\` previamente planejado, mantendo \`autoSync=false\` e \`existingResources=adopt\`. O sync \`exe-dc0eenek1f9s738ssumg\` atualizou \`pandora-orchestrator-staging\` com \`ORCH_ENV=production\` e \`ORCH_DATABASE_URL\` via \`fromDatabase\` apontando ao PostgreSQL gratuito isolado \`pandora-orchestrator-staging-db\`. Não criou outro banco nem publicou strings de conexão. Deploy \`dep-dc0ehhss728c73a0p920\` concluiu em \`live\` em 2026-11-21T00:42:51Z e registrou HTTP 200 em \`GET /health\`. O lifespan \`get_service()\` instancia \`Store\`, que abre a conexão e cria/verifica as tabelas antes de completar a inicialização. Isso demonstra que a nova configuração passou pelo bootstrap, mas **não** valida persistência após reinício com dados de teste nem integração autenticada de Cloud para Orchestrator. O Cloud mantém \`PANDORA_ORCH_SITE_TOKENS_JSON={}\`: sem credenciais de cliente. Todas as ferramentas mutáveis e MCP permanecem fechadas. Documentação revisada: [runbook commit 830e60a](https://github.com/raphaelsousapereira/pandora-corpo/commit/830e60a3c6e56e4c09f85acea2a727285a09315f). Pendente: smoke autenticado com dois tenants sintéticos, isolamento/expiração/revogação e persistência PostgreSQL real em múltiplas execuções; preservar senhas fora do GitHub.


### 2026-11-20 · Deploy final de documentação do Orchestrator staging

Depois da publicação do runbook em `pandora-corpo` (commit `830e60a3c6e56e4c09f85acea2a727285a09315f`), o Render executou novo deploy automático `dep-dc0ei915efls73e2ndog`, status **live** em 2026-11-21T00:44:09Z. O último código de aplicação é o mesmo; a alteração foi documental. Continua pendente teste autenticado fim a fim e persistência de registros sintéticos após reinício. A configuração do Cloud permanece sem tokens de tenant, logo nenhuma execução de minitarefas foi liberada.


## 2026-11-20 · ORQUESTRADOR-2026-004 · Homologação PostgreSQL e ponte HTTPS real com dois tenants sintéticos

- **Projeto/componente, ambiente, revisão e fuso:** \`pandora-corpo/packages/pandora_orchestrator\` (PR [#161](https://github.com/raphaelsousapereira/pandora-corpo/pull/161), merge \`66abfd604473e51a3819aa7729b1cf21494671b7\`, reforço de log commit \`1a5ee70cbd3f57ffdd42b470fa015dddf452a7d1\`); \`pandora-cloud\` (PR [#264](https://github.com/raphaelsousapereira/pandora-cloud/pull/264), merge \`83d7608824cf419c4650b26b3dc0301e2e91980f\`). Render staging Oregon; 20/11/2026 America/Sao_Paulo.
- **Pedido e aceite:** executar homologação com dois clientes sintéticos, provar persistência entre deploys e comunicação HTTPS efetiva Cloud → Orchestrator com isolamento org/site, sem expor acesso de clientes reais nem permitir ações externas, com código versionado no GitHub.
- **Memória consultada antes da execução:** \`8f6f4f4bc62f139637ecd058614aefa1dc7f3f6a\`, casos ORQUESTRADOR-2026-001/002/003. Primeiro teste: recusa total a SQLite com \`ORCH_ENV=production\`; em seguida dois tenants sintéticos com \`Orchestrator.plan\` e tentativas cruzadas.
- **Implementação e testes locais:** módulo opt-in \`orchestrator/staging_smoke.py\` (flag \`ORCH_STAGING_SMOKE_ENABLED=1\` no lifespan), grava dois registros de agente de teste em PostgreSQL e marcador \`orch_staging_smoke\`, bloqueia acesso a org/site alheio e relê dados em sessão independente. Suíte **97 passed**, 3 testes novos para fixture, recusa SQLite e recusa development. No Cloud, \`orchestrator_bridge_smoke.py\` testa endpoint real health/version, prévia de plano em leitura, busca do conhecimento já aprovado (vazia nos tenants sintéticos) e tentativa de acesso remoto com token do outro tenant. **6 testes locais passados com dublês**, além da validação real nos logs Render. Nenhum GitHub Actions nem API paga executado.
- **Evidência real do PostgreSQL:** deploy \`dep-dc0ekn1it1ds73ajq0d0\` live; log em 2026-11-21T00:49:56Z: \`STAGING_SMOKE\` passou com \`bootstrap_count=2\`, \`cross_tenant=blocked\`, \`cross_site=blocked\`, \`db=postgresql\`, \`restart_persistence_verified=true\`. Após revogação das credenciais de teste, novo deploy \`dep-dc0eomc0916s73bg1930\` chegou a **live** em 2026-11-21T00:57:46Z, log \`STAGING_SMOKE\` (00:57:42Z) passou novamente com \`bootstrap_count=4\`. O contador persistiu entre reinicializações reais.
- **Evidência HTTPS Cloud → Orchestrator:** deploy Cloud \`dep-dc0emuvv243c73b3oo1g\` chegou a **live** em 2026-11-21T00:56:23Z; log 00:56:23Z \`CLOUD_ORCH_BRIDGE_SMOKE\`: \`status=passed\`, \`synthetic_tenants=2\`, \`read_only_preview=passed\`, \`approved_knowledge_search=empty_as_expected\`, \`cross_tenant_http=blocked\`. São solicitações **HTTPS reais entre serviços Render**, sem dados de clientes; os endpoints externos mutáveis não foram exercitados.
- **Revogação e estado operacional final:** após testes, o mapa de tokens sintéticos do Cloud foi zerado \`PANDORA_ORCH_SITE_TOKENS_JSON={}\` e \`PANDORA_ORCH_BRIDGE_SMOKE_ENABLED=0\`. O backend substituiu hashes temporários por uma única entrada de hash inacessível (token de origem descartado, prazo curto), mantendo a validação fail-closed de produção. O deploy Cloud de limpeza \`dep-dc0eon3m8hqs73e9bjr0\` chegou a **live** em 2026-11-21T00:58:51Z e respondeu \`/health\` 200. Não existe token de site real liberado. A bridge de diagnóstico permanece habilitada, porém endpoints por org/site sem token retornam bloqueio. MCP remoto, crawling e agentes Claude Code/Codex permanecem desativados.
- **Limitações explícitas:** não foi realizado teste de UI com usuário autenticado e site real, nem OAuth delegado fim a fim, nem medição de custo de ferramentas, nem carga de produção; banco gratuito staging expira conforme condições da Render. Não afirmar que agentes estão liberados para clientes ou que ferramenta executa ações reais em produção.
- **Próxima ocorrência:** consultar este ID, conferir /health de ambos deploys, revisar flags/expiração, provisionar credencial limitada somente após seleção de site/consentimento e escopo operacional, homologar MFA e controle de acesso de usuário real sem habilitar execução externa.
- **Evidências:** [PR #161](https://github.com/raphaelsousapereira/pandora-corpo/pull/161), [PR #264](https://github.com/raphaelsousapereira/pandora-cloud/pull/264), [código do teste Cloud](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/pandora_cloud/orchestrator_bridge_smoke.py), [Render Orchestrator](https://dashboard.render.com/web/srv-dc0ed6t9el5s73dro6og), [Render Cloud](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg).


### 2026-11-20 · Fechamento ORQUESTRADOR-2026-004 — probe staging desligado

Após o smoke multicliente, \`ORCH_STAGING_SMOKE_ENABLED\` foi desativado no serviço Render (valor \`0\`) para interromper gravações sintéticas em reinicializações futuras. O deploy final \`dep-dc0epre1egvs73fmu5ng\` chegou a **live** em 2026-11-21T01:00:21Z, com GET \`/health\` 200 nos logs. A configuração de Cloud já havia sido limpa de tokens sintéticos e de \`PANDORA_ORCH_BRIDGE_SMOKE_ENABLED\`, com deploy \`dep-dc0eon3m8hqs73e9bjr0\` **live** às 00:58:51Z. Resultado definitivo da rodada: persistência PostgreSQL e isolamento de duas organizações/site fictícios verificados, leitura HTTPS de prévia e conhecimento aprovada em staging, credenciais temporárias retiradas, nenhum acesso de cliente real habilitado. O banco de staging e o código dos probes permanecem para futuras homologações autorizadas, mas os probes estão desligados por padrão.


## 2026-11-20 · ORQUESTRADOR-2026-005 · Consentimento explícito e liberação leitura por site

- **Projeto, componente, ambiente e fuso:** \`pandora-cloud\` / \`pandora_cloud/orchestrator_bridge_routes.py\`; Render Cloud serviço \`srv-dbmj0q6417fc73eobvtg\`; America/Sao_Paulo.
- **Pedido e aceite observável:** seguir da homologação sintética para habilitação **controlada** de sites reais, com autorização individual do cliente, MFA, escopos restritos, diagnóstico e revogação; não adicionar nenhum cliente automaticamente e não habilitar execução autônoma.
- **Memória consultada:** GitHub \`pandora-memoria-tecnica/MEMORIA.md\` no commit \`dc8054ba0176387ab5bb93089badb09cf84c1513\`; casos ORQUESTRADOR-2026-001/002/003/004. **Primeiro teste distintivo:** administrador da plataforma sem grant ativo de cliente não pode provocar chamada remota.
- **Causa encontrada:** \`CloudService.audit_block(..., admin=True)\` permite auditoria administrativa sem autorização prévia do cliente, o que **não** é critério suficiente para encaminhar contexto a um Orchestrator externo; a ponte anterior confundia esses dois níveis de permissão.
- **Correção:** PR [#265](https://github.com/raphaelsousapereira/pandora-cloud/pull/265), merge \`f58c9b9afd55cb38bb0169c622b42a27f3a3ae13\`. Ponte exige registro SQL ativo \`access_grants.scope='audit'\` para o mesmo org/site, não arquivamento, \`platform_admin\` com MFA concluído, token de backend único para o par e \`PANDORA_ORCH_REAL_TENANTS_ENABLED=1\`. O admin da plataforma não pode criar esse consentimento em nome do cliente. Rota local GET \`/api/admin/orquestrador/habilitacao\` retorna apenas estado de elegibilidade, sem rede/segredos. \`/integracao\` não consulta servidor remoto se falta algum gate; \`/preview\` e \`/conhecimento\` falham fechados com 403/409. Mantidas as verificações de JWT e de tenant/site já existentes.
- **Testes:** 15 cenários **locais** com FastAPI/SQLite sintético e bridge mock em \`tests/test_orchestrator_release_gate.py\` passaram, incluindo consentimento ausente/revogado, MFA pendente, site arquivado, escopo errado, organização/site incorretos, token ausente, flag desligada e sucesso somente leitura; \`compileall\` local passou. Imports da suíte legada \`tests/test_orchestrator_bridge.py\` corrigidos (antes \`orch_bridge_stage\`, pacote não presente no repo), fixture adaptada com grant SQL. Não foi executada suíte completa do Cloud nem jornada com cliente real.
- **Implantação e bloqueios:** a flag \`PANDORA_ORCH_REAL_TENANTS_ENABLED=0\` foi explicitamente escrita no Render Cloud, mantendo a funcionalidade externa **desativada para clientes reais** por padrão. Os tokens sintéticos anteriores permanecem revogados/limpos. Verificar o último deploy Render antes de declarar release do código; não afirmar que cliente real está conectado.
- **Efeitos colaterais e reversão:** rotas administrativas que antes mostravam health sem consentimento passam a mostrar apenas prontidão local, por proteção de dados. Reverter novos acessos com \`PANDORA_ORCH_REAL_TENANTS_ENABLED=0\` (sem necessidade de remover código); para desligar inclusive o diagnóstico usar \`PANDORA_ORCH_BRIDGE_ENABLED=0\`. Em revogação de grant ativo, a próxima requisição é negada antes de enviar conteúdo remoto; também revogar hash e token.
- **Próxima ocorrência:** operador seleciona site com consentimento audit ativo concedido pelo cliente, valida MFA e utiliza a rota local de elegibilidade. Somente então gera token temporário único (hash backend, segredo apenas env Cloud) e habilita o rollout por ambiente de piloto. Testar UI autenticada, revogação e observabilidade no primeiro site real antes de falar em liberação de produção.
- **Evidências:** [PR #265](https://github.com/raphaelsousapereira/pandora-cloud/pull/265), [runbook](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/ORCHESTRATOR_READ_ONLY_ROLLOUT.md), [Render Cloud](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg).


### 2026-11-20 · Fechamento ORQUESTRADOR-2026-005 — Cloud live e rollout desligado

PR #265 integrado no commit \`f58c9b9afd55cb38bb0169c622b42a27f3a3ae13\`. Depois de configurar \`PANDORA_ORCH_REAL_TENANTS_ENABLED=0\`, o último deploy Render do \`pandora-cloud\` \`dep-dc0f02r7tt1c73bc65vg\` chegou a **live** em 2026-11-21T01:16:47Z. Logs: \`Application startup complete\`, \`GET /health\` HTTP 200 após inicialização. A implementação está publicada no serviço real, com o gate de clientes reais deliberadamente desligado. Os 15 testes focados foram locais com serviços simulados; não há evidência de jornada com identidade de cliente real. Nenhum cliente foi conectado e nenhuma permissão externa foi ampliada.
