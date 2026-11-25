# Memória técnica acumulativa da PANDORA

Índice adicional: ENCONTROS-2026-001 · reuniões · gravação longa · reenvio · decisões · fatos editoriais.

Índice pesquisável: AUDIO-2026-001 · Dora · microfone oculto · Render live · Whisper · feature flag · Chromium · transcrição em português. Acrescente os próximos projetos ao índice e ao final do arquivo.

Índice Porto Alegre em Dados: PORTOALEGRE-2026-001 · WPVibe 526 · SSL de site novo · PORTOALEGRE-2026-002 · ITBI · m² por bairro · efeito de composição · prédios novos · PORTOALEGRE-2026-003 · Wikimedia Commons 429 · Banco de Imagens da Prefeitura · PORTOALEGRE-2026-004 · tempo de leitura inflado · gerador não determinístico · CSV CRLF · GEO-2026-001 · GEO Score editorial · falsos negativos do medidor · PORTOALEGRE-2026-005 · LicitaCon DMAE 54907 · contrato não localizado · valor de comunicado divergente.

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


## 2026-11-20 · ORQUESTRADOR-2026-006 · Seleção de piloto Negócio no Mapa

- **Projeto, componente, ambiente:** \`pandora-cloud/pandora_cloud/orchestrator_pilot.py\`, \`orchestrator_bridge_routes.py\`; piloto domínio público \`seonegocionomapa.com.br\`; Render serviço \`srv-dbmj0q6417fc73eobvtg\` com liberação de tenants reais **desligada**.
- **Pedido/aceite:** usuário escolheu explícita e exclusivamente Negócio no Mapa como primeiro piloto de homologação. Não liberar os outros sites nem presumir concessão de auditoria Cloud a partir da conexão WordPress.
- **Memória consultada:** revisão \`e8917d3c4232ffaae1df2f7f1921388e2c0f3138\` e casos ORQUESTRADOR-2026-004/005. Primeiro teste distintivo: localizar host EXATO no Cloud SQLite e verificar \`access_grants.scope='audit'\` ativo e não arquivamento. Uma credencial WordPress WPVibe não prova esse vínculo.
- **Descoberta:** WPVibe confirmou \`https://seonegocionomapa.com.br\` conectado e usuário WP administrador; não é evidência de consentimento \`access_grants\` dentro do Cloud. O conector Render disponível não lê a base SQLite de Cloud diretamente.
- **Implementação:** PR [#266](https://github.com/raphaelsousapereira/pandora-cloud/pull/266), merge \`46c482e1422e173878f2cc3f82f392db992a0b69\`. Novo \`inspect_pilot\` consulta apenas SQLite local por hostname exato (+ alias www), localiza IDs org/site, consentimento audit ativo, revogação e site arquivado, e recusa selecionar automaticamente se houver dois cadastros do mesmo domínio. Nova rota administrativa autenticada com MFA \`GET /api/admin/orquestrador/piloto\` retorna os IDs somente ao administrador; diagnóstico de startup com \`PANDORA_ORCH_PILOT_DOMAIN\` registra apenas domínio, número de registros e consentimento (sem IDs ou tokens).
- **Verificação local:** 15 testes do módulo Python passaram, incluindo domínio falso, subdomínio, consentimento ausente/revogado, site arquivado e ambiguidade. Não executar WordPress nem API de código.
- **Configuração:** \`PANDORA_ORCH_PILOT_DOMAIN=seonegocionomapa.com.br\` e \`PANDORA_ORCH_REAL_TENANTS_ENABLED=0\` configurados no Render Cloud; deploy(s) disparados, status definitivo e leitura da elegibilidade devem ser confirmados antes de provisionar credencial. \`PANDORA_ORCH_SITE_TOKENS_JSON={}\` continua sem site real. Nenhum grant criado pelo administrador ou alteração no WordPress.
- **Próximo passo conforme evidência:** se existir exatamente um cadastro com audit grant ativo, prover token exclusivo org/site no Cloud e SHA-256 limitado ao par no staging (não em GitHub), sem habilitar outros sites, e testar chamada read-only com sessão admin MFA. Se não houver cadastro/grant, registrar bloqueio preciso e não contornar o consentimento.


### 2026-11-20 · Resultado ORQUESTRADOR-2026-006 — domínio duplicado no Cloud

Render Cloud em deploy \`dep-dc0fhcu43dis73dcbgr0\`, commit \`46c482e1422e173878f2cc3f82f392db992a0b69\`, registrou às 2026-11-21T01:52:51Z: \`ORCH_PILOT_READINESS {"consent_ready": false, "domain": "seonegocionomapa.com.br", "site_count": 2, "unique_site": false}\`. O serviço completou inicialização e reportou GET \`/health\` 200; deploy \`live\` em 01:52:53Z. **Interpretação restrita:** existem dois registros do hostname (www aceito) e a condição de candidato único falhou; \`consent_ready=false\` é resultado da ambiguidade, **não demonstra** que nenhum dos registros tenha grant ativo. Não habilitar todos, não presumir a organização correta e não provisionar token por domínio sem identificar \`org_id/site_id\` da conta aprovada. A rota protegida \`GET /api/admin/orquestrador/piloto\` já retorna os dois pares e flags para o administrador com MFA concluído, sem enviar dados ao Orchestrator. \`PANDORA_ORCH_REAL_TENANTS_ENABLED=0\` permaneceu desligado; nenhuma credencial criada. Precisa haver escolha fundamentada do cadastro correto por administrador após visualizar os dois registros (ou prova de que somente um reúne consentimento audit e vínculo correto). Não mesclar/excluir cadastros automaticamente.

## 2026-11-20 · ORQUESTRADOR-2026-007 · Evidence Evaluation v0.5.0

- **Projeto, revisão, ambiente:** `pandora-corpo/packages/pandora_orchestrator` v0.5.0; commit merge `afe2037ca0b0a797bb9a5b2cf47d1560dcbb883b`, PR #162; Render staging `srv-dc0ed6t9el5s73dro6og`; fuso America/Sao_Paulo.
- **Pedido e aceite:** avançar próxima fase do Orchestrator após estudo de arquiteturas de agentes, priorizando avaliação de soluções/evidências e preservação dos controles multi-tenant. O site-piloto `seonegocionomapa.com.br` permanece bloqueado devido a dois cadastros no Cloud.
- **Memória consultada antes de editar:** revisão `9bf2c99136abe754048001023588729e1a4bca4d`, casos ORQUESTRADOR-2026-004/005/006. Primeiro teste distintivo: a mesma fonte, quando não concedida ao agente ou fora de org/site, não pode produzir relatório de atribuição positivo.
- **Diagnóstico:** Knowledge Network v0.4.2 dispõe de fontes aprovadas por hash, grants por agente e consulta lexical, mas não possui avaliação explícita da integridade da atribuição de uma alegação por fontes específicas. A adoção de modelo semântico não seria suportada por teste de precisão/erros.
- **Correção implementada:** novo `orchestrator/evidence_evaluator.py` (consulta somente leitura das fontes aprovadas+granted, hash, expiração, agente ativo e escopo), `POST /v1/evidence/preview` com 1–5 fontes; retorna sobreposição lexical, elegibilidade e `human_review_required=true`, com `not_verified` explícito para veracidade, implicação semântica e independência. Nenhuma promoção automática nem chamada web/IA/MCP/código. Atualizada versão para 0.5.0, testes e documentação.
- **Testes:** 15 testes locais direcionados `pytest` passaram usando SQLite em memória + SQLAlchemy, incluindo isolamento, não divulgação de URLs não autorizadas, fonte pendente/revogada/expirada/adulterada, agente inativo, inputs inválidos e ausência de efeitos colaterais; `compileall` OK. Não foi executada suíte integral nem teste de endpoint HTTP real/PostgreSQL remoto nesta entrega.
- **Implantação:** PR [#162](https://github.com/raphaelsousapereira/pandora-corpo/pull/162) integrado em `afe2037ca0b0a797bb9a5b2cf47d1560dcbb883b`. Render iniciou deploy `dep-dc0g8760tbcc73b041n0`; conferir status e logs antes de marcar live. Os flags `ORCH_KNOWLEDGE_WEB_ENABLED=0`, `ORCH_CODE_ENABLED=0`, `ORCH_MCP_READ_ENABLED=0` permanecem desligados.
- **Limite/risco:** sobreposição lexical **não prova verdade**; parecer de atribuição é suporte à revisão, sem score de confiança científica. Não ativar usuários reais; evitar fonte enviesada e duplicação de citações. Retificar esta entrada com status do Render quando comprovado.
- **Próxima ocorrência:** testar API real com pares sintéticos, avaliar falsos positivos com corpus anotado por humanos antes de considerar motor semântico, verificar duplicidade do site piloto Cloud e consentimento específico.
- **Links:** [PR #162](https://github.com/raphaelsousapereira/pandora-corpo/pull/162), [documentação v0.5](https://github.com/raphaelsousapereira/pandora-corpo/blob/main/packages/pandora_orchestrator/docs/EVIDENCE_EVALUATION_V050.md).

### 2026-11-20 · ORQUESTRADOR-2026-007 — staging v0.5.0 live

Render `srv-dc0ed6t9el5s73dro6og` confirmou deploy `dep-dc0g8760tbcc73b041n0` com commit `afe2037ca0b0a797bb9a5b2cf47d1560dcbb883b`, status **live**, finalizado em 2026-11-21T02:39:21Z. Logs registraram `Application startup complete`, `GET /health` 200 e URL `https://pandora-orchestrator-staging.onrender.com`. Confirma implantação operacional de v0.5.0, **não** comprova decisão semântica, fluxo autenticado da nova rota nem acurácia de respostas reais. Sem alteração de flags ou concessão a clientes. O piloto do Negócio no Mapa ainda possui dois registros Cloud e não foi liberado.

## 2026-11-20 · ORQUESTRADOR-2026-008 · Operador de ferramentas SaaS e aplicação WPVibe

- **Projeto/revisão/ambiente:** `pandora-cloud`, PR [#267](https://github.com/raphaelsousapereira/pandora-cloud/pull/267), squash merge `6ca98559f9383f5e67bb38e5e5011113576eb81e`, serviço Render `srv-dbmj0q6417fc73eobvtg`, horário Brasil.
- **Pedido e critério observável:** agente deve abrir ferramentas SaaS, executá-las sob autorização, absorver relatórios e encaminhar soluções para WPVibe. Aceite parcial deste estágio: usar os nove adaptadores **já implementados** pelo AgentRuntime, com relatório consolidado, proposta vinculada a hash, aprovação e adaptador WordPress de troca textual precisa; **não** afirmar execução de todas as APIs do SaaS nem escrita real sem OAuth e teste de ponta a ponta.
- **Memória consultada:** revisão do GitHub `MEMORIA.md` correspondente aos casos ORQUESTRADOR-2026-005/006/007; primeiro teste: administrador sem consentimento audit do cliente ou sem allowlist org/site não pode iniciar agente ou escrever no WordPress.
- **Causa/arquitetura:** `pandora_cloud/agent_runtime.py` já possui `AGENTS` com nove agentes, `start` e `status`; `site_memory.py` agrega medições e cruzamentos. `executiva_wpvibe.py` possui OAuth 2.1 independente do ChatGPT, mas não executor arbitrário de tools/call. Não criar proxy genérico ou reutilizar conexão do ChatGPT.
- **Mudança:** novos módulos `pandora_cloud/orchestrator_operator.py`, `orchestrator_operator_routes.py`, `orchestrator_operator_wpvibe.py`, documentação e testes. Rotas administrativas para catálogo, execução de adaptadores, relatório individual e consolidado, proposta, aprovação e aplicação WPVibe. Cada operação exige admin verificado/MFA concluído, grant `audit` ativo, site não arquivado; execução exige `PANDORA_ORCH_OPERATOR_ENABLED=1` **e** `PANDORA_ORCH_OPERATOR_ALLOWED_PAIRS_JSON` contendo par explícito org/site. Discovery pago tem gate extra. WPVibe tem gate separado; o endpoint é fixo `POST /wpvibe/v1/content/edit` para substituir uma ocorrência específica de `post_content`, sem códigos arbitrários, WP-CLI nem exclusões. Após qualquer resultado incerto, **não retentar automaticamente**, manter necessidade de reconciliação humana.
- **Testes:** `pytest` direcionado em módulos isolados com SQLite e `httpx.MockTransport`: **22 passed** (20 iniciais mais 2 cenários de memória consolidada escopada). `compileall` dos módulos originais OK. Foram removidos stubs globais de `sys.modules` antes do merge para não poluir a suíte integral. A suíte integral do Cloud, publicação WPVibe real e OAuth do SaaS **não** foram executadas.
- **Estado operacional:** os quatro controles de execução foram gravados explicitamente como seguros no Render Cloud: `PANDORA_ORCH_OPERATOR_ENABLED=0`, `PANDORA_ORCH_OPERATOR_ALLOWED_PAIRS_JSON={}`, `PANDORA_ORCH_OPERATOR_PAID_ENABLED=0`, `PANDORA_ORCH_WPVIBE_APPLY_ENABLED=0`. Merge disparou deploy(s) do Cloud. A situação final precisa ser confirmada por logs e status de deploy. O domínio `seonegocionomapa.com.br` tem dois registros e ainda não está habilitado.
- **Próxima ocorrência:** escolher `org_id/site_id` correto do piloto, verificar consentimento em Cloud, configurar OAuth separado do SaaS no WPVibe, fazer ensaio real de prévia e leitura, obter autorização exata para uma alteração pontual e rodar pós-verificação/reversão. Expandir os demais módulos do SaaS por manifestos/adaptadores tipados e testes; não prometer "qualquer endpoint" sem controle.
- **Evidência:** [PR #267](https://github.com/raphaelsousapereira/pandora-cloud/pull/267), [runbook](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/ORCHESTRATOR_OPERATOR_WPVIBE_V060.md).

### 2026-11-20 · Fechamento ORQUESTRADOR-2026-008 — Cloud live e ações fechadas

O merge Cloud v0.6 PR #267 (`6ca98559f9383f5e67bb38e5e5011113576eb81e`) e atualização de flags produziram dois deploys Render: `dep-dc0ggnjrjlhs739ujjbg` **live** em 2026-11-21T02:58:13Z, depois `dep-dc0ggppit1ds73al00ug` **live** em 2026-11-21T03:00:00Z. Logs após o último deploy retornaram `GET /health` **200** em 03:00:04Z e 03:00:05Z, sem stack traces nos registros consultados. Flags persistidas: operador nativo 0, allowlist vazia {}, execução de Discovery paga 0, aplicação WPVibe 0. Isso confirma implantação do **código fechado**, não uma execução ao vivo dos agentes nem alteração WordPress. WPVibe OAuth SaaS precisa de autenticação independente e homologação do tool schema para validar `rest_api` MCP; o conector ChatGPT não fornece token ao SaaS. Primeiro piloto `seonegocionomapa.com.br` continua com dois cadastros no Cloud, sem escolha autorizada.

## 2026-11-21 · ORQUESTRADOR-2026-009 · Evolução supervisionada dos próprios motores

- **Escopo/revisão/ambiente:** PANDORA Cloud `raphaelsousapereira/pandora-cloud`, PR [#269](https://github.com/raphaelsousapereira/pandora-cloud/pull/269), squash merge `0c22c79ff88815687f02f208eb69bb3495c12a97`, Render `srv-dbmj0q6417fc73eobvtg`, fuso America/Sao_Paulo.
- **Pedido/aceite:** quando um motor, ferramenta ou função falhar, Orchestrator sugere atualização de implementação, testes e arquivo afetado, apresenta para aprovação, e após aprovação prepara correção no GitHub. Não confundir sugestão e PR com código seguro, merge ou deploy já executados.
- **Memória consultada antes da edição:** `MEMORIA.md` revisão `196aa7691f328ced0f29d71a21ad18c7ffaae7be`, especialmente ORQUESTRADOR-2026-005/006/007/008. **Primeiro teste distintivo:** uma falha não confirmada (run ainda rodando), pertencente a outro org/site ou acompanhada de hash alterado nunca pode gerar mudança nem PR.
- **Implementação:** módulos `pandora_cloud/orchestrator_improvements.py`, `orchestrator_improvements_routes.py`, `orchestrator_improvements_github.py`; página administrativa `/admin/orquestrador/melhorias?org=<hex32>&site=<hex32>`, JS de review e doc `docs/ORCHESTRATOR_SELF_IMPROVEMENT_V070.md`. Verificador busca até 20 execuções nativas anteriores do AgentOperator e retorna **somente falhas reais**, classificadas conservadoramente em timeout/429/autenticação/contrato de dados/desconhecida. Apresenta hipótese, regressões recomendadas e mapeamento de motor para arquivo explícito. Registro de proposta de patch exato exige hash do relatório ainda atual; revisão e aprovação exigem hash imutável, administrador com MFA concluído, `access_grants scope=audit` vigente e org/site válido. Sem evidência de causa raiz verificada.
- **Ação após aprovação:** apenas com `PANDORA_ORCH_SELF_REPAIR_PR_ENABLED=1`, token independente de GitHub App no servidor, `PANDORA_ORCH_OPERATOR_ENABLED=1` e org/site presente na allowlist. O adaptador verifica arquivo e ocorrência exata do trecho original, abre branch `orchestrator-fix/<proposal_id>` e PR **draft**, nunca merge, deploy ou edição WordPress. Resultado incerto da criação bloqueia tentativa repetida e exige reconciliação.
- **Segurança/operação:** `PANDORA_ORCH_SELF_REPAIR_PR_ENABLED=0` configurada explicitamente no Render após merge; nenhuma credencial GitHub App foi criada. Flags de operador, allowlist e WPVibe permanecem desligadas. O Negócio no Mapa segue com dois cadastros no Cloud; não ativar um deles por domínio. O catálogo inicial abrange apenas os nove motores AgentRuntime existentes, não qualquer função do SaaS.
- **Testes:** suite local direcionada de **19/19 passed**, `compileall` OK. MockTransport simula as cinco chamadas API do GitHub e valida PR rascunho, não merge, conflito em arquivo-fonte e token ausente. Cobertura de MFA, isolamento org/site, hash adulterado, consentimento via guard injetado, aprovação concorrente, retry negado e busca de falhas. Não foi executada a suíte integral do Pandora Cloud, nem criação real de PR por token OAuth SaaS, nem validação da UI autenticada com cliente real.
- **Implantação:** PR merge confirmado; atualização de ambiente Render disparou deploy com gate fechado. **Verificar status do último deploy e health antes de declarar live**. O painel é um fluxo de revisão e não uma IA capaz de fabricar correções corretas sozinha; a proposta de código ainda requer autor humano ou coding agent autorizado, acompanhado de teste e revisão.
- **Links:** [PR #269](https://github.com/raphaelsousapereira/pandora-cloud/pull/269), [documentação](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/ORCHESTRATOR_SELF_IMPROVEMENT_V070.md), [Render Cloud](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg).

## 2026-11-21 · PAA-2026-001 · Perguntas brutas não eram plano de ação nem Google PAA

- **Projeto, componente, revisão e ambiente:** pandora-cloud, engine/PANDORA_PAA.py 1.3.0 e engine/paa_action_plan.py 1.0.0; serviço Render pandora-cloud srv-dbmj0q6417fc73eobvtg. Fuso America/Sao_Paulo. PR #268, merge 18fb321e79ff38d7a5636bae571b2bb615c03f50.
- **Pedido e aceite observável:** pesquisar Google PAA e concorrentes internacionais, converter lacunas em plano com urgência/prazos/fontes e publicar com sucesso no Render. Workspace My Workspace confirmado pelo usuário. Não habilitar operadores, escrita WordPress ou IA paga para cumprir a entrega.
- **Memória consultada:** main 8c8ee6d; busca PAA/pauta/prazos sem caso anterior; AUDIO-2026-001 aplicável à distinção entre deploy live, health e jornada autenticada. Primeiro teste: pergunta com nome da concorrente e parágrafo com múltiplas perguntas devem ficar fora das tarefas editoriais.
- **Sintoma demonstrado:** a comparação retornava perguntas extraídas, com prioridades por frequência entre concorrentes. Um só concorrente produzia prioridade baixa para todas; isso não mede urgência de negócio. O levantamento podia incluir chamada genérica, promoção da marca concorrente e parágrafo composto. Não era cronograma nem resultado observado de Google PAA.
- **Pesquisa e arquitetura:** Google Search Central descreve perguntas relacionadas/featured snippets sem divulgar algoritmo integral de seleção. AlsoAsked organiza PAA observado no Google; Semrush pesquisa temas/perguntas; Frase combina oportunidades com ações de conteúdo. Fontes e limites em docs/PAA_ACTION_PLAN.md. Não alegar implementação do algoritmo do Google. Nenhum scraping Google ou novo provedor pago foi adicionado.
- **Correção:** build_plan puro com quatro fases (validar antes de executar 0–7 dias; melhorar páginas 8–30; preparar conteúdo 31–60; reavaliar 61–90), contadas após aprovação, não urgência comercial medida. ID estável, pergunta/fonte, destino quando disponível, função sugerida, dependências e condição de conclusão. Relevância comercial é heurística de ordenação, sem volume, tráfego, ROI ou promessa de ranking. Excluir tarefas sem URL válida, marca concorrente específica, trechos compostos, chamadas genéricas e IDs ambíguos, preservando evidência bruta. Sem comparação válida, não inferir tarefas editoriais. Página associada sem trecho é explicitamente inconclusiva.
- **Integração:** engine inclui action_plan; service.detail recalcula planos históricos sem gravar a coleta original; painel/assistente/PDF e JSON/CSV privados incluem plano. Mantém autenticação/CSRF/escape e CSV contra fórmulas. Nenhuma tarefa Executiva ou publicação criada automaticamente. Bootstrap executa plano inconclusivo sintético e valida quatro fases antes de servir o portal.
- **Tentativas e verificação:** 30 testes motor/plano/API/exportações/histórico, 30 chat/rotas incluindo autenticação herdada, 16 limites/PAA página única/plano de visibilidade/memória, 40 regressões JS e 160 casos sintéticos aprovados; navegador PAA com motor real e sites sintéticos aprovado, incluindo fases, urgência, exportações, móvel e XSS. Dados reais já coletados Fraga e Santos × Camila Chalon: plano no snapshot com 24 sugestões elegíveis e três exclusões; sem nova escrita de produção. O benchmark sintético não é avaliação independente nem medida 530–850. Suíte integral não executada; sessão real de cliente no SaaS não exercitada.
- **Publicação:** o primeiro pedido de merge usou SHA abreviado e foi rejeitado com 422 sem alteração; repetido uma vez com SHA completo e PR integrado. Auto-deploy dep-dc0gmartqb8s73dfo66g iniciou a partir de 18fb321. Deploy confirmado live em 2026-11-21T03:10:17Z. Verificação no domínio app.geoaicitations.com: /health 200, paa-studio.js e paa-assistant.js 200 com SHA-256 idêntico aos arquivos do commit publicado, export/plano sem sessão 401. Logs de runtime confirmam esses códigos. Não houve teste autenticado real em produção. Verificação de bootstrap não apareceu no filtro de logs; não alegar mensagem observada. Memória atualizada após pull de db49b0f, incluindo ORQUESTRADOR-2026-009 de outra entrega concorrente; preservar flags e alterações alheias. Python padrão não tinha requests; verificação HTTP feita com urllib sem instalar dependências.
- **Recuperação e próxima ocorrência:** reverter PR #268 se houver regressão; mudança é aditiva e não migra tabelas. Confirmar commit servido e abrir relatório histórico, comparar plano/JSON/PDF e testar troca de relatório. Fonte site_questions e google_paa_observed=false devem permanecer; dados SERP futuros exigem provedor autorizado, consulta, local/idioma, data e URLs observadas. Não marcar cliente/região como relevante sem revisão humana.
- **Evidências:** https://github.com/raphaelsousapereira/pandora-cloud/pull/268 ; https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/PAA_ACTION_PLAN.md ; https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0gmartqb8s73dfo66g .

### 2026-11-21 · Retificação PAA-2026-001 — deploy concorrente concluído

O deploy subsequente do Orquestrador `dep-dc0gmphit1ds73al4130`, commit `0c22c79ff88815687f02f208eb69bb3495c12a97` descendente do merge PAA, ficou **live** em 2026-11-21T03:12:14Z. Diff confirmou preservação do motor/plano e JS do PAA. Nova verificação do domínio após esse deploy: health 200 e ambos os JS com conteúdo idêntico ao commit PAA. Não houve sessão autenticada de cliente.

### 2026-11-21 · Fechamento ORQUESTRADOR-2026-009 — Render Cloud live

Render `srv-dbmj0q6417fc73f5k9o0` (atenção: este é um ID do workspace, não um ID de serviço; o serviço correto é `srv-dbmj0q6417fc73eobvtg`) confirmou deploy `dep-dc0gmphit1ds73al4130` no commit `0c22c79ff88815687f02f208eb69bb3495c12a97`, **live** em 2026-11-21T03:12:14Z. Health checks `GET /health` HTTP 200 às 03:12:18, :23, :27 e :28 UTC, sem Traceback/ERROR nos últimos logs consultados. O endpoint de aprovação da v0.7 e o painel administrativo estão publicados; o código é fail-closed, pois `PANDORA_ORCH_SELF_REPAIR_PR_ENABLED=0` e não há credencial GitHub App instalada para essa função. **Não houve criação de PR de reparo real, merge, deploy de alteração proposta ou escrita WordPress.** O piloto Negócio no Mapa permanece bloqueado devido aos dois registros de site. Próxima validação: registrar par de site correto, testar login/MFA real, consentimento e GitHub App OAuth com revisão humana do primeiro patch, sem liberar operações arbitrárias.

### 2026-11-21 · Retificação VOZ-2026-001 — leitura local nos dois portais

O leitor prioriza português brasileiro e vozes locais identificadas como premium ou aprimoradas. Segmenta frases antes do limite de comprimento. Testes simulados verificaram preferência local, primeira pausa e integridade do texto; qualidade acústica em iPhone ainda não aferida. Render confirmou live no Cloud (dep-dc0iedrtqb8s73dh8840, e5c5132) e portal de arquitetura (dep-dc0iefvv243c73b66740, 99e6251). Sem contratação de síntese ou serviço adicional.

## 2026-11-21 · ANEXOS-2026-HTML · HTML e legendas na conversa da Dora

Ampliados o seletor e o leitor local para HTML/HTM e legendas SRT/VTT. O texto HTML é extraído sem abrir links; as legendas preservam a referência temporal. PR https://github.com/raphaelsousapereira/pandora-cloud/pull/270. Anexos ativados no Render; deploy dep-dc0l0oqd217c73faer5g live no commit 18bbfb5. Incluídos testes de regressão. Execução Python e upload autenticado não verificados nesta sessão.

## 2026-11-21 · ORQUESTRADOR-2026-010 · Studio Conversacional de Missões v0.8

- **Projeto, componente, revisão e ambiente:** `raphaelsousapereira/pandora-cloud`; PR [#272](https://github.com/raphaelsousapereira/pandora-cloud/pull/272) integrado no commit `0d1ed9b1d26518cb0398f266468c8d8c3944e7b5`; Render Cloud serviço `srv-dbmj0q6417fc73eobvtg`, workspace `tea-d85u03bbc2fs73f5k9o0`; 21/11/2026, fuso America/Sao_Paulo.
- **Pedido e critério observável:** usuário aprovou Orchestrator Ω Studio Conversacional, integrado à UI e missões existentes, com histórico, contexto, aprovação, ferramentas e relatórios, e pediu encerrar só após confirmação `live` do Render.
- **Memória consultada antes de alterar:** revisão `1a4bd76ad9fc716ec1ce3d9db349b03348d7b26a`, casos ORQUESTRADOR-2026-005 a 009 e regras de `AGENTS.md`/`MODELO.md`. Primeiro teste distintivo: uma missão criada para um site não pode ser consultada/aprovada/executada em outro site nem após revogação do consentimento audit. Nenhum registro do domínio duplicado Negócio no Mapa deve ser autoeleito.
- **Implementação:** `pandora_cloud/orchestrator_studio.py`, `orchestrator_studio_routes.py`, `orchestrator_studio_ui.html`, `orchestrator_studio.css`, `orchestrator_studio.js`; menu Ferramentas → Agentes → Orchestrator Ω e retorno pela UI de Melhorias. Página `/admin/orquestrador/studio` exige administrador + MFA; seletor distingue explicitamente pares org/site com consentimento audit; SQLite persiste threads, mensagens e etapas; recebe objetivo em linguagem natural e monta plano **determinístico** de até seis tarefas por catálogo nativo, exige aprovação vinculada a SHA-256. Aprovar não executa. Cada etapa inicia somente após clique e `AgentOperator.guard(... execute=True)`; execução incerta impede retentativa automática. Relatórios individuais e `SiteMemory` consolidado escopados. Frontend usa textContent, CSP restritiva e layout responsivo. Não há interpretação generativa livre, voz, anexos ou autonomia arbitrária na v0.8; UI declara esses limites.
- **Segurança:** não modificou `PANDORA_ORCH_OPERATOR_ENABLED=0`, `PANDORA_ORCH_OPERATOR_ALLOWED_PAIRS_JSON={}`, `PANDORA_ORCH_WPVIBE_APPLY_ENABLED=0` nem `PANDORA_ORCH_SELF_REPAIR_PR_ENABLED=0`; sem token de site habilitado, execução paga, escrita WordPress, GitHub PR automático ou alteração de clientes. Gate de Discovery pago requer confirmação explícita e gate original do operator.
- **Testes de implementação:** `pytest -q tests/test_orchestrator_studio.py`: **9 passed** com SQLite temporário, FastAPI TestClient, criação/leitura de missão, MFA, revogação, isolamento, hash, aprovação concorrente, execução única, relatório do operador, interfaces estáticas. `python -m compileall` e `node --check` passaram. `tools/ci_local.py` completo não executado (sem checkout integral do repo no runtime); teste de navegador com login real ainda pendente.
- **Publicação e evidência do runtime:** PR #272 merge `0d1ed9b1d26518cb0398f266468c8d8c3944e7b5`; deploy Render `dep-dc0le6rtqb8s73djmvh0` **live**, finalizado 2026-11-21T08:34:10.484787Z; logs confirmaram `Application startup complete`, `GET /health` HTTP 200 e URL primária `https://app.geoaicitations.com`. Isto confirma implantação e saúde do serviço, **não** teste autenticado de toda a conversa em produção.
- **Recuperação e próxima ocorrência:** rollback do PR #272 se regressão; nunca habilitar ferramentas por site sem consentimento individual. Para próximo estágio, testar UI real com usuário administrador/MFA, pacote completo de CI, custos e limites de AIGateway antes de criar chat semântico; validar cadastros duplicados do primeiro piloto.
- **Links:** [PR #272](https://github.com/raphaelsousapereira/pandora-cloud/pull/272), [documentação do Studio](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/ORCHESTRATOR_STUDIO_V080.md), [Render live](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0le6rtqb8s73djmvh0).

### 2026-11-21 · PAA-2026-001 — recuperação por página

Durante o deploy às 05h17 de Brasília, o processo foi encerrado antes de gravar o relatório. PR #273 do pandora-cloud preserva no SQLite dados por página, links descobertos, sites concluídos e orçamento global, e exibe o limite efetivo no formulário. Testados 13 casos de serviço, inclusive reinício com continuação, mais coleta sintética de 150 páginas por domínio (300 no total). Render live em dep-dc0lfnfb9u1s73blq9k0 no commit 7ff1797. Execuções antigas já interrompidas não tinham checkpoint.

## 2026-11-21 · ORQUESTRADOR-2026-011 · Hierarquia 10 líderes × 10 especialistas

- **Projeto, ambiente e revisão:** `raphaelsousapereira/pandora-cloud`, PR [#274](https://github.com/raphaelsousapereira/pandora-cloud/pull/274), squash merge `548a087cb64eb4e7cc8f2eeb6b37a2bca91c2e12`, Render Cloud `srv-dbmj0q6417fc73eobvtg`, em 21/11/2026 America/Sao_Paulo.
- **Pedido e aceite observável:** 1 principal, 10 suborquestradores departamentais, até 10 especialistas por líder (capacidade de 100). Principal não pode dirigir tarefas a especialistas sem passar pelo domínio. Ver capacidade verdadeira, autorizada e parcialmente provisionada; não declarar agentes autônomos em funcionamento sem runtime correspondente.
- **Memória consultada antes da edição:** `MEMORIA.md` revisão `c6a02fc5ef29ad14d321bbb36b14240fc3f3ba5a`, casos ORQUESTRADOR-2026-009/010 e regras de `AGENTS.md`/`MODELO.md`. **Primeiro teste distintivo:** principal → especialista direto, líder → especialista de outro departamento e ferramenta sem adaptador recebem bloqueio; acesso a catálogo de outro site exige consentimento/MFA.
- **Diagnóstico:** Studio v0.8 usa planejamento determinístico de no máximo seis etapas e nove adaptadores nativos. Não havia árvore departamental com 100 vagas, nem visualização de líderes ou verificação explícita do encaminhamento. Não existem 100 trabalhadores de IA provisionados.
- **Implementação:** novo `pandora_cloud/orchestrator_hierarchy.py` com dez domínios definidos (GEO/SEO; observabilidade; entidades; conteúdo; citações; competição; publicação; UX/CRO; engenharia; governança), dez vagas estáveis por líder, mapeamento unívoco de **nove ferramentas nativas** para especialistas e **91 vagas planned**. `route_tool` recusa ferramentas desconhecidas e `authorize_hop` valida `principal → leader → specialist`. `orchestrator_studio.py` acrescentou `hierarchy`, relatórios de execução agregados por líder nas missões e checagem de rota antes de lançar cada passo (sempre usando o `AgentOperator` preexistente). Novo GET `/api/admin/orquestrador/studio/hierarquia` protegido pelo consentimento atual e MFA. Studio mostra líderes de forma recolhível, contagem e disponibilidade; sem fingir 100 agents ativos.
- **Testes e correções:** suite `pytest -q tests/test_orchestrator_studio.py tests/test_orchestrator_hierarchy.py` reconstruída sobre os módulos efetivos do Studio: **20 passed** (12 Studio + 8 hierarquia). Primeiro ensaio detectou dois erros no PR: arquivo de teste buscava o módulo no diretório errado e `route_tool` referia-se a `catalog` antes de inicialização; corrigidos antes do merge. `python -m compileall` do backend e `node --check` (JS prévia local) sem erro. Suite completa `tools/ci_local.py` **não executada**, porque não há checkout integral no ambiente; UI autenticada real ainda não exercitada.
- **Deploy real:** Render deploy `dep-dc0lhtrbc2fs7380i460`, commit `548a087cb64eb4e7cc8f2eeb6b37a2bca91c2e12` **live** em 2026-11-21T08:42:08.999791Z. Logs registraram inicialização completa, `GET /health` HTTP 200 e domínio principal `https://app.geoaicitations.com`. Não é prova de trabalho automático dos 100 slots.
- **Segurança:** mantém `PANDORA_ORCH_OPERATOR_ENABLED=0`, allowlist de sites vazia, e gates separados WPVibe, GitHub e self-repair. O domínio piloto Negócio no Mapa continua com dois cadastros no banco Cloud, sem escolha automática. Não amplia clientes, não executa agentes não registrados, não gera custos novos.
- **Próxima ocorrência:** adicionar manifestos e testes de cada especialista novo; depois, se necessário, desenvolver workers departamentais separados e políticas de fila/limites e observabilidade, sem romper as permissões. Validar tela e consentimento reais com admin MFA.
- **Links:** [PR #274](https://github.com/raphaelsousapereira/pandora-cloud/pull/274), [documentação](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/ORCHESTRATOR_HIERARCHY_V090.md), [Render live](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0lhtrbc2fs7380i460).

## 2026-11-21 · ORQUESTRADOR-2026-012 · Website Factory Ω: blueprint, QA e rascunhos WordPress supervisionados

- **Projeto/ambiente/revisão:** `raphaelsousapereira/pandora-cloud` PR [#275](https://github.com/raphaelsousapereira/pandora-cloud/pull/275), merge `aa2d59a90b929903b36b011a1ea7306c571f3792`, Render serviço `srv-dbmj0q6417fc73eobvtg`, deploy `dep-dc0lr1jrjlhs73a32n40`, 2026-11-21 America/Sao_Paulo.
- **Pedido/critério:** desenvolver com conclusão e teste por fase uma capacidade de construir ou reconstruir sites completos usando ferramentas SaaS, D1–D10, WPVibe, QA, PR merge e Render live; usuário quer relatório final, não planos soltos. **Limite do aceite:** a versão implementada gera projetos/artefatos, mas ainda não cria site completo real nem publica cliente; liberar produção sem um site autorizado seria falso/indevido.
- **Memória consultada antes da mudança:** `MEMORIA.md` revisão `f17b41b851c4a734e496098dd0f1648d6ea103d1`, especialmente ORQUESTRADOR-2026-008/009/010/011, e `AGENTS.md`, `MODELO.md`. Primeiro teste distintivo: dois registros do mesmo domínio no Cloud NÃO compartilham autorização de projeto; o motor não pode exportar nem criar rascunho no registro errado.
- **Descoberta:** `AgentOperator` já integra nove ferramentas nativas e dispõe de `audit` grant e MFA; OAuth WPVibe do SaaS é separado do conector ChatGPT. O adaptador anterior só substituía fragmento textual aprovado, não gerava páginas inteiras. WPVibe site_info conectado confirmou a versão atual do plugin 1.24.1 no Negócio no Mapa com acesso administrador, Abilities API 19 abilities (a conexão ChatGPT não concede autorização SaaS). Nenhum cadastro duplicado foi selecionado.
- **Correção/arquivos:** novos `pandora_cloud/orchestrator_website_factory.py`, `orchestrator_website_factory_routes.py`, `orchestrator_website_factory_wpvibe.py`, `orchestrator_website_build.py`, `orchestrator_website_contract.py` e interface `.html/.css/.js`, testes, documentação `docs/ORCHESTRATOR_WEBSITE_FACTORY_V1.md`. Integração `portal.py`, `workspace_ui.py` e link no `orchestrator_studio_ui.html`.
- **Fases implementadas:** briefing com escopo; arquitetura de até doze páginas; design de prévia acessível e estática; conteúdo sem invenção factual; QA determinístico; staging de **rascunho** WordPress condicionado a aprovações de fases + hash SHA256 + OAuth SaaS independente + allowlist + `PANDORA_ORCH_WEBSITE_DRAFT_ENABLED=1` (padrão **0**); release final **bloqueado** até backup, URLs/redirects, homologação e rollback reais. O principal recebe relatórios e a matriz D1–D10 com cem vagas, **só nove adaptadores nativos mapeados**, sem simular cem trabalhadores. Exportação de site estático navegável ZIP com index, páginas, navegação, CSS, robots.txt `Disallow`, `noindex` e manifest SHA-256; arquivo tem aviso explícito de não publicação.
- **Segurança:** isolamento org/site, MFA, revogação, sites arquivados, texto HTML escapado, CSP de sandbox em preview, cada etapa aprovada por hash e ordem, respostas WPVibe ambíguas nunca acionam repetição e retorno só é sucesso se contiver `status=draft`, `id` inteiro e slug idêntico. Nenhuma alteração em cliente ou WPVibe realizada. Flags de operador e WPVibe existentes preservadas. D10 veta release.
- **Testes executados:** `pytest -q tests/test_orchestrator_website_factory.py`: **19 passed** em SQLite temporário/FastAPI TestClient com mocks de OAuth, aderência do ZIP, XSS, troca de site, consentimento, MFA, fontes malformadas, slugs duplicados, hash, QA, idempotência e resposta incerta; `python -m compileall` e `node --check` sem falhas. Suíte global do Cloud e navegador autenticado com usuário real **não executados**. Mock não comprova criação de página real via WPVibe.
- **Publicação:** PR #275 squash merged na main (`aa2d59a90b929903b36b011a1ea7306c571f3792`), Render deploy `dep-dc0lr1jrjlhs73a32n40` confirmado **live** em 2026-11-21T09:02:05.740681Z, logs `Application startup complete`, `GET /health` HTTP 200. Web tool não abriu domínio público nesta sessão (erro de acesso), portanto **não** alegar jornada de interface real autenticada. A saúde do serviço não demonstra publicação WordPress.
- **Causa ainda incerta/próxima ocorrência:** escolher cadastro site correto e consentimento audit, estabelecer OAuth WPVibe específico no SaaS, criar um rascunho real em site de homologação, revisar render visual, medir performance e checar redirects/backups; implementar editor de páginas, publicação final com rollback e workers reais por departamento. Rollback de PR #275 caso a inicialização ou rotas apresentem regressão; checar logs de Render e GET /health primeiro.
- **Links:** [PR #275](https://github.com/raphaelsousapereira/pandora-cloud/pull/275), [manual](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/ORCHESTRATOR_WEBSITE_FACTORY_V1.md), [Render deploy](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0lr1jrjlhs73a32n40).


### 2026-11-21 · Retificação PAA-2026-001 — atividade durante a coleta

O contador do PAA publicava só domínio e total. O PR [#279](https://github.com/raphaelsousapereira/pandora-cloud/pull/279) registra tentativas HTML por URL e distingue falha de página lida com zero perguntas. O painel traz as últimas 12 e o relatório web filtra o inventário. Cinco casos isolados do fetch e inicialização repetida do SQLite passaram. Merge 6d5d5b7, Render dep-dc0qbgfb9u1s73bqf3pg live. Jornada autenticada com sites reais pendente. Para regressão verificar page_activity no detalhe do job, a leitura aceita e o resultado HTTP antes de investigar o componente visual.


## 2026-11-21 · ORQUESTRADOR-2026-013 · Catálogo central de capacidades Ω 1.0

- **Projeto, componente e ambiente:** `raphaelsousapereira/pandora-cloud`; `pandora_cloud/capability_registry.py`, AgentOperator, Mission Studio e Website Factory; Render `srv-dbmj0q6417fc73eobvtg`, America/Sao_Paulo.
- **Pedido e aceite:** criar inventário compartilhado de todas as capacidades encontráveis no SaaS, registrar permissão/estado/custo/evidência, evitar que ferramentas bloqueadas sejam propostas como prontas e publicar no Render. Descoberta não deve conceder acesso remoto ou permitir endpoints arbitrários.
- **Memória consultada antes da edição e publicação:** `MEMORIA.md@a0f42ba9fef9704555bf655dd52570bcab592b43`, casos ORQUESTRADOR-2026-006, 008, 010, 011 e 012; `AGENTS.md` e `MODELO.md`. **Primeiro teste distintivo:** operador com flag global ligada mas sem o par exato na allowlist não pode marcar nenhum adaptador nativo como executável; ferramenta MCP jamais é executável por herança de credencial.
- **Sintoma/causa:** nove adaptadores realmente executáveis no AgentRuntime, dezenas de páginas no diretório Workspace e 37 contratos MCP eram catálogos distintos. O Mission Studio escolhia adaptadores só pelo registro, não pela elegibilidade de execução. A Website Factory mostrava contagem registrada sem refletir as flags por site. A árvore de 100 especialistas contém 91 vagas planejadas, não trabalhadores ativos.
- **Correção:** PR [#281](https://github.com/raphaelsousapereira/pandora-cloud/pull/281) squash merge `f7af6197c238f417898a45c463e7a86b03840a8f`. `capability_registry.py` monta registro limitado de origem AgentRuntime, `workspace.TOOLS`, `workspace_ui.AGENT_GROUPS`/`NAV` e `mcp_remote.TOOLS`. Estado por ferramenta `ready`, `blocked`, `confirmation_required`, `manual_only`, `separate_auth`; reporta custo, permissões, canal, última solicitação (sem fingir sucesso) e motivo de bloqueio. Operador obtém o catálogo sob admin+MFA+grant audit+org/site, calcula allowlist, flags e estado. Studio planeja só adaptadores autorizados, hierarquia marca indisponíveis, Factory recebe resumo da mesma fonte. Interface `/admin/orquestrador/capacidades` requer seleção explícita do par e não executa ações; API `/api/admin/orquestrador/capacidades` e aliases de Studio/Factory.
- **Segurança:** MCP fica em `separate_auth`, UI em `manual_only`; nenhum executor genérico, nenhuma nova flag habilitada, nenhum token transferido, nenhuma alteração WPVibe, cadastro duplicado não unificado. Seleção de site com MFA e grant audit rechecados. Discovery pago continua com confirmação por operação. A recomendação externa usou documentos oficiais do Google Agent Registry, MCP e A2A, sem implementar falsamente um Google Registry remoto.
- **Verificações:** invariantes estáticas de arquivos, rotas, política e presença dos testes verificadas no branch, PR sem conflito de merge. Adicionados `tests/test_orchestrator_capability_registry.py` e regressões de operador/Studio, mas **não foram executados os testes Python/JS nesta sessão**: runs de GitHub Actions `44924801411`, `44924801346`, `44924801342` falharam antes de expor etapas (`steps=null`). O build Render instala dependências mas não executa essa suíte. Não confundir isso com sucesso de execução autenticada de cada ferramenta.
- **Publicação:** Render deploy [`dep-dc0sf3jbc2fs7387qttg`](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0sf3jbc2fs7387qttg) do commit exato `f7af6197c238f417898a45c463e7a86b03840a8f` confirmado `live` em 2026-11-21T16:33:52Z; logs `Application startup complete`, `GET /health` 200 e URL pública `https://app.geoaicitations.com`. Isto prova versão implantada e serviço saudável, **não** prova que o catálogo respondeu a um administrador autenticado nem que sites WordPress foram alterados.
- **Risco/recuperação e próxima ocorrência:** testar em sessão admin/MFA `GET /api/admin/orquestrador/capacidades?org=...&site=...` para um site autorizado, inspecionar status e uma solicitação real nativa (sem custo) sob permissão; testar revogação e cadastro duplicado com testes locais. Em erro na interface/rotas, conferir logs e rollback do merge #281. Piloto `seonegocionomapa.com.br` segue com dois registros e `consent_ready=false`; não escolher nem liberar automaticamente.
- **Evidência:** [PR #281](https://github.com/raphaelsousapereira/pandora-cloud/pull/281), [manual](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/ORCHESTRATOR_CAPABILITY_REGISTRY_V100.md), [Render Live](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0sf3jbc2fs7387qttg).


## 2026-11-21 · ORQUESTRADOR-2026-014 · Autorização individual de adaptadores por site

- **Projeto/ambiente/revisão:** `raphaelsousapereira/pandora-cloud`, PR [#282](https://github.com/raphaelsousapereira/pandora-cloud/pull/282) squash merge `8fcad65acb7fa33f7171f180c8a6070863c500c9`; Render Cloud `srv-dbmj0q6417fc73eobvtg`; fuso America/Sao_Paulo.
- **Pedido e aceite observável:** usuário pediu autorizar o acesso do Orquestrador às capacidades após implantar o Catálogo Central. Criar mecanismo real de autorização/revogação por ferramenta e cadastro exato, sem abrir todos os clientes ou executar ferramentas automaticamente; configurar política e confirmar Render `live`.
- **Memória lida antes de alterar:** `MEMORIA.md@68118ba58fa2649d227203e21fc5654bdf7754f9`, casos ORQUESTRADOR-2026-013 e 006/008/010/011/012; `AGENTS.md` consultado. **Primeiro teste distintivo:** com operador e allowlist ativados, `native_executable=0` sem concessão individual; autorizar um único nativo deve tornar só ele elegível e revogar deve bloquear a próxima execução.
- **Causa/correção:** inventário v1.0 distinguia 9 adaptadores, UI e MCP, mas aprovação por site dependia somente da allowlist Render. PR #282 adicionou `orchestrator_native_tool_approvals` (chave org/site/agent_tool, administrador, referência, timestamps, revogação) e eventos de auditoria; `AgentOperator.authorize_native`/`revoke_native`, bloqueio no `launch` e estado no registro `capability_registry.py`. Endpoints POST `/api/admin/orquestrador/capacidades/autorizar` e `/revogar` exigem administrador autenticado e MFA; autorização exige consentimento audit ativo do cliente, site pre-allowlisted e justificativa de 20–600 caracteres. UI exige seleção do cadastro exato e confirmação específica por ferramenta. O Mission Studio e a Website Factory consultam o mesmo estado de elegibilidade.
- **Política aplicada em Render:** merge de variáveis sem substituir as outras: `PANDORA_ORCH_OPERATOR_ENABLED=1` e `PANDORA_ORCH_TOOL_APPROVALS_REQUIRED=1`. A allowlist preexistente NÃO foi modificada e nenhum par org/site nem grant de cliente foi inserido por esta operação. Isso torna a aprovação nativa individual obrigatória no operador, inclusive quando o par já é elegível no Render. Discovery exige ainda política de custo e confirmação por chamada; MCP e WordPress não receberam novo acesso.
- **Verificação:** inspeção estática de rotas, políticas, testes e código; JavaScript da interface validado pelo parser JS do runtime do conector (`new Function`). Testes Python adicionados em `tests/test_orchestrator_operator.py` e `tests/test_orchestrator_capability_registry.py`, mas **não executados nesta sessão** por ausência de runner ativo no canal conectado; GitHub Actions tem histórico de workflows falhando antes das etapas. Nenhum login real/MFA de usuário foi usado, nenhum clique de autorização e nenhum job real foi executado; por isso o estado do consentimento no banco permanece não comprovado.
- **Publicação real:** primeiro deploy do merge `dep-dc0sq2p5efls73egc0rg` **live** em 2026-11-21T16:57:30Z. Segundo deploy de configurações `dep-dc0sq859k2hc73duopdg` **live** em 2026-11-21T16:59:22Z com commit exato `8fcad65acb7fa33f7171f180c8a6070863c500c9`, logs `Application startup complete`, `GET /health` HTTP 200 repetidos e URL primária `https://app.geoaicitations.com`. Isso confirma bootstrap, não autorização individual concluída.
- **Limites e próxima ocorrência:** abrir `/admin/orquestrador/capacidades`, selecionar site que possua consentimento audit e conste da allowlist Render, escrever justificativa e autorizar individualmente cada nativo gratuito; testar resposta autenticada e execução controlada. O domínio `seonegocionomapa.com.br` permanece duplicado (`site_count=2`, `unique_site=false`, `consent_ready=false` nos logs) e não deve ser eleito sem confirmação do registro correto. Se houver regressão, revisar o PR #282 e status/erros no Render; reverter flag somente mediante revisão das implicações de política.
- **Referências:** [PR #282](https://github.com/raphaelsousapereira/pandora-cloud/pull/282), [manual](https://github.com/raphaelsousapereira/pandora-cloud/blob/main/docs/ORCHESTRATOR_CAPABILITY_REGISTRY_V100.md), [deploy Render live](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc0sq859k2hc73duopdg).

## 2026-11-22 · DIAGNOSTICO-2026-001 · Testes do main não rodavam e três regressões chegaram ao Render

- **Projeto, componente, ambiente, revisão e fuso horário:** `pandora-cloud` main `1823931` (Render `dep-dc0tguu7bikc739rife0` live com esse commit), `pandora-corpo` main `c015489`; janela 2026-11-20 02:00 UTC → 2026-11-22 02:00 UTC; horários em UTC.
- **Pedido e critério observável de aceite:** diagnóstico técnico dos avanços no main das últimas 48 h, apontando falhas e lacunas com evidência reproduzível.
- **Memória consultada antes da mudança (commit e IDs):** `MEMORIA.md@74f5723`; ORQUESTRADOR-2026-005…014, PAA-2026-001, VOZ-2026-001. Primeiro teste distintivo: `python tools/ci_local.py` no main e, para cada falha, o mesmo teste no commit anterior à janela (`2b454de`).
- **Sintoma, ponto de reprodução, comportamento esperado e observado:** as entradas 010–014 registram "suíte completa não executada"; todos os runs do GitHub Actions no main falham em cerca de 3 s, antes das etapas. Rodando o CI local: unittest (1104 s), PAA, biblioteca, imprensa e grafismo foram aprovados; os testes de navegador falharam.
- **Causa demonstrada, evidência e causas ainda incertas:**
  1. `44e8931`/`a9c21bb` (#246, planos): removido o desvio que deixava o administrador incluir site extra em org Premium. `MAX_SITES[cliente]=1` agora vale para todos. Oito testes de navegador quebram com `Conflict: O plano desta conta cobre 1 site(s)` (visibility_weekly, plugins, velocidade, entidade, youtube, enderecos, integracoes). Todos passavam em `2b454de`. Em produção, org Premium com mais de um site não ganha site novo nem pelo administrador.
  2. #277/#278 (Ajuda no login): a regra `@media(max-width:600px){body:has(input:focus) .lg-ajuda{position:static…}}` muda o layout no foco. O clique em "Entrar com link" é perdido e `tests/browser_login.py` falha de forma determinística. Sem essa linha, o teste passa. Passava em `2b454de`.
  3. Nove dos 13 arquivos `tests/test_orchestrator_*.py` (106 funções pytest) não rodam no CI: `unittest discover` ignora funções pytest, e o passo pytest do `ci.yml` lista só 4 arquivos. Ao rodar todos, há 192 aprovados e 3 falhas: `test_orchestrator_bridge::test_revoked_audit_never_reaches_orchestrator`, que espera 403 enquanto a rota devolve 200 com `connected:false` (sem chamada remota, teste desatualizado), e dois de `test_orchestrator_operator` (rotas autorizar/revogar), com o dublê SQLite sem `check_same_thread=False`. As rotas de autorização do #282 seguem sem teste executável.
  4. Plano `personalizado` (#246) é tratado como não-Premium em `escriba_jobs._full_allowed` (`== plans.CLIENT`) e na tela de consentimento OAuth (`mcp_remote_routes.py:76`).
  5. Já existia antes da janela: `tests/browser_workspace.py` também falha em `2b454de` (timeout em `#as-new`, botão removido/realocado).
- **Cobertura real, simulações, falhas de ambiente e limites da conclusão:** pip-audit (requirements.txt e locks do forge) não encontrou vulnerabilidades. As falhas de axe/pip-audit no primeiro `--sem-instalar` foram de ambiente. axe-core passou depois de instalado. Pacote `pandora-corpo/packages/pandora_orchestrator`: 114 testes aprovados, mas nenhum workflow do corpo os executa. Não houve login real, MFA, nem consulta a logs Render (workspace Render não selecionado nesta sessão).
- **Lacunas de processo:** 22 commits empurrados direto no main do Cloud sem PR, o que contraria a regra de CI local do `CLAUDE.md`. Sem entrada de memória para #277/#278/#280/#283, Facets e corpo #163 (worker editorial que publica no WordPress com `status=publish` quando GEO ≥ 950, sem revisão humana do texto). `PANDORA_ORCH_TOOL_APPROVALS_REQUIRED` tem padrão `0` no código (fail-open), mitigado só pela variável no Render.
- **Próxima ocorrência:** antes de merge, rode `PANDORA_CHROMIUM=<chrome> python tools/ci_local.py` e `python -m pytest tests/test_orchestrator_*.py`. Se um teste de navegador falhar com `Conflict` de plano, conferir `core.create_site` e `MAX_SITES`.

### 2026-11-22 · Retificação DIAGNOSTICO-2026-001 — limite de sites era intencional; correções 1 e 2
- O item 1 não era regressão do código: `test_planos.test_um_site_por_plano` registra que "administrador também respeita o limite contratado" (#246). Os sete testes de navegador estavam desatualizados e agora usam o plano `personalizado`. Org Premium real com mais de um site precisa trocar de plano.
- Commits em `pandora-cloud` no branch `claude/main-technical-diagnosis-psy4k9`:
  - `b6f6887`: a Ajuda do login fica `visibility:hidden` no foco, sem sair de `position:fixed`.
  - `31765ec`: os testes de navegador com vários sites usam o plano `personalizado`.
  - `39b7c1e`: o CI roda `pytest tests/test_orchestrator_*.py` (195 aprovados), corrige os 3 testes e ganha a guarda `tests/test_cobertura_pytest.py`.
- CI local do branch: 10 jobs aprovados e 2 com falha, as duas também presentes no main:
  - `browser_workspace.py` já falhava antes da janela.
  - O audit `pandora-ux` exige o "Ícone visível nos atalhos de GEO Score e NAP" no Assistente, atalhos removidos em `0bde734` sem atualizar `scripts/ux_cloud_local.py`.

### 2026-11-22 · Retificação DIAGNOSTICO-2026-001 — auditoria de UX e browser_workspace corrigidos
- **Auditoria `pandora-ux`:** a regra "Ícone visível nos atalhos de GEO Score e NAP" (#254) mirava um `svg` com `aria-hidden="true"`. O `COLLECT` de `packages/pandora_ux/pandora_ux/visual.py` descarta esse elemento, então a regra nunca era medida e reprovava em 320 e 390 px.
  - A captura mostrava os ícones na tela: o defeito era a medição, não a interface.
  - Correção: PANDORA UX 1.5.0 com a opção `decorative: true` em regras `target` (padrão `false`), usada nos manifestos `cloud` e `cloud-cliente`.
  - Recebe também a correção de `filter(visible)`, que passava o índice como segundo argumento.
- **`browser_workspace.py`:**
  - "Nova conversa" (`#as-new`) saiu do topo em `0bde734`; o teste agora recarrega a página, já que a conversa não é guardada.
  - O menu do celular virou "Mais opções de navegação" em #254; o teste usa `#pc-menu`.
  - Lacuna de produto em aberto: não há mais botão para começar outra conversa sem recarregar, e o handler de `#as-new` em `assistente-app.js` ficou sem uso.
- **Commits:** `2637744` e `fc71ba5` no PR #284.
- **CI local em `fc71ba5`:** 12 aprovados, 0 falhas, 6 pulados (caminhos não tocados).
- **Próxima ocorrência:** se uma regra visual vier como "Nenhum conjunto completo de elementos visíveis foi medido" com o elemento aparecendo na captura, conferir `aria-hidden`, `hidden` e `inert` no seletor antes de mexer no CSS.

## 2026-11-22 · ORQUESTRADOR-2026-015 · Auditoria: escrita no WordPress usa ferramenta só de leitura

- **Projeto, componente, ambiente, revisão:** `pandora-cloud` main `1823931` (Render live); `orchestrator_operator_wpvibe.py`, `orchestrator_website_factory_wpvibe.py`, `orchestrator_website_wordpress.py`.
- **Pedido:** auditar o Orquestrador, avaliar se a fábrica de agentes valeu a pena e o que falta para funcionar agora.
- **Memória consultada:** ORQUESTRADOR-2026-001…014, DIAGNOSTICO-2026-001.
- **Causa demonstrada:** os três adaptadores de escrita chamam a ferramenta MCP `rest_api` com `method` POST/PUT. O contrato atual do WPVibe diz que `rest_api` é GET-only e que POST, PUT e DELETE vão por `rest_api_write`.
  - Afeta: criar rascunho (Factory), aplicar edição exata (operador), stage, publicação e rollback (release v2).
  - Os testes usam dublês e nunca exercitaram o schema real. A pendência "homologar tool schema" de ORQUESTRADOR-2026-008 se confirma como bloqueio.
- **Estado operacional:**
  - Flags: `OPERATOR_ENABLED=1`, `TOOL_APPROVALS_REQUIRED=1`, allowlist de pares vazia, `REAL_TENANTS=0`, WPVibe, rascunho, release e self-repair em 0.
  - O piloto `seonegocionomapa.com.br` tem 2 cadastros (`consent_ready=false`).
  - Nenhuma execução real de cliente aconteceu.
- **Outras constatações:**
  - O Studio é determinístico: escolhe ferramentas por palavras-chave e as mensagens não têm interpretação por IA.
  - A hierarquia 10×10 tem 100 vagas, das quais só 9 têm um motor nativo.
  - Os jobs rodam em threads dentro de 1 instância com SQLite em disco de 1 GB: um deploy interrompe a execução (mesmo mecanismo do PAA-2026-001).
  - Existem dois orquestradores: o do Cloud, que executa, e o serviço do `pandora-corpo` (durable executor/PostgreSQL), que o Cloud só consulta em leitura e com a liberação desligada.
- **Correção sugerida (não aplicada):** trocar para `rest_api_write` (POST/PUT) e manter `rest_api` para GET. Exigir `rest_api_write` no `tools/list` e criar um teste de contrato com o schema real.
- **Próxima ocorrência:** antes de ativar WordPress, rodar `tools/list` na conexão OAuth do SaaS e conferir o `inputSchema` de `rest_api` e `rest_api_write`.

### 2026-11-22 · Correção ORQUESTRADOR-2026-015 — escritas por rest_api_write
- **Commit:** `205194c` no PR raphaelsousapereira/pandora-cloud#284.
- **Contrato:** `pandora_cloud/wpvibe_contract.py` escolhe `rest_api` para GET e `rest_api_write` para POST/PUT/DELETE, e valida os argumentos contra o `inputSchema` de `tools/list`. Se o contrato do WPVibe mudar, a chamada falha com 409 antes de sair.
- **Onde foi aplicado:** edição exata do operador, rascunho da Factory e `WordPressExecutor.request` (stage, publicação e rollback). Este último guarda o catálogo por 5 min.
- **Teste:** `tests/test_orchestrator_wpvibe_contract.py`, com fixture `tests/fixtures/wpvibe_rest_tools.json` copiado do schema do servidor MCP oficial em 2026-11-22.
  - Cada chamada é validada com `jsonschema`, e o servidor falso recusa escrita em `rest_api`.
  - No código antigo, 7 de 8 testes falham; no novo, a suíte do Orquestrador tem 203 aprovados.
- **Limite:** não houve chamada real ao WPVibe, porque falta a conexão OAuth do SaaS e um site de homologação.
- **Primeiro passo na ativação:** com a conexão OAuth do Studio Executiva, rodar `tools/list` e comparar os schemas de `rest_api` e `rest_api_write` com o fixture. Se divergirem, atualizar o fixture e o teste antes de ligar `PANDORA_ORCH_WEBSITE_DRAFT_ENABLED` ou `PANDORA_ORCH_WPVIBE_APPLY_ENABLED`.

### 2026-11-23 · Retificação PAA-2026-001 — HTTP 200 com conteúdo montado por JavaScript

No PANDORA Cloud, o inventário registrava HTTP 200 e o PAA mostrava zero páginas medidas. A memória consultada foi `862edbd45fd809569c24b711764e01d10b06348c`. O primeiro teste comparou o mesmo documento antes e depois do Chromium: três URLs públicas passaram de zero texto extraído para texto legível. O HTML de origem trazia um contêiner vazio e carregava o corpo via JavaScript.

O [PR #296](https://github.com/raphaelsousapereira/pandora-cloud/pull/296) reaproveita o Chromium do Raio-X para contêineres de aplicativo sem texto ou perguntas suficientes. O PAA entrega ao navegador o documento já coletado, bloqueia novas navegações e valida domínio, robots.txt e orçamento para recursos. O HTML renderizado alimenta o extrator e a descoberta de links. A suíte PAA/Raio-X rodou 44 casos, com três pulados. Uma rodada local com marca sintética e concorrente real mediu três páginas e três perguntas em uma amostra limitada a cinco URLs. Houve aviso de conteúdo insuficiente em parte da amostra. A rodada usou o proxy de saída do ambiente. Ainda falta validar o fluxo autenticado publicado. Se houver regressão, reverter o PR. Para recorrências, comparar o corpo recebido e o DOM antes de alterar as regras de extração.

### 2026-11-23 · Retificação PAA-2026-001 — correção publicada no Render

Com autorização de publicação, o PR #296 foi integrado à main em `7143849391657e7b6a592e152a1aafcdfad6db1b`. A memória consultada foi `061ebf96a955a36fb81e1db8d8424fd17fbc4711`. O GitHub Actions não iniciou a avaliação por limite de orçamento. A publicação usou a verificação local anterior, incluindo regressões do assistente e avaliação por família, e a comparação dos quatro arquivos do PR com o merge. O deploy automático [dep-dc25bjl9el5s73bf61gg](https://dashboard.render.com/web/srv-dbmj0q6417fc73eobvtg/deploys/dep-dc25bjl9el5s73bf61gg) ficou live às 12h05, horário de Brasília. O domínio publicado respondeu HTTP 200 em /health e HTTP 401 em /api/admin/paa sem sessão. A primeira consulta aos logs após a ativação retornou zero erros de aplicação. O navegador abriu a tela de login, então uma nova rodada autenticada em produção segue pendente.

## 2026-11-23 · AJUDA-2026-001 · vídeos de ensino nas ferramentas bloqueados pela CSP

- **Projeto, componente e revisão:** PANDORA Cloud, ajuda das ferramentas (`pandora_cloud/aprenda.py`, `static/aprenda-mais.js`/`.css`, middleware `SecurityHeaders` em `portal.py`). Base `7143849`, branch `claude/aprenda-mais-videos-block-19ed0l`. Fuso: UTC.
- **Pedido e aceite:** bloco "Aprenda mais" dentro das ferramentas com vídeos de ensino hospedados no YouTube como não listados. A capa deve mostrar a imagem do vídeo, e não apenas um botão.
- **Memória consultada:** `912c1a6`. Não havia caso sobre vídeo incorporado. VIDEO-2026-001 se aplica parcialmente: manter JS/CSS fora do HTML por causa da CSP.
- **Sintoma e causa:** `brand.csp()` começa com `default-src 'none'` e não tem `frame-src`/`media-src`, então qualquer iframe do YouTube é bloqueado. O portal também envia `Referrer-Policy: no-referrer`, e o player do YouTube recusa tocar sem a origem. Por fim, a ajuda existente (`tool_help.py`) só aceitava a ferramenta `schema`.
- **Armadilha evitada:** várias rotas acrescentam `; frame-src 'self'` à CSP. Se `csp()` passasse a declarar um `frame-src` próprio, o navegador usaria só a primeira diretiva e as prévias em iframe dessas rotas quebrariam. Por isso a liberação **funde** origens na diretiva existente (ou herda de `default-src`), e só nas páginas com vídeo cadastrado.
- **Correção:**
  - O catálogo `help_content/aprenda.pt-BR.json` mapeia o caminho da página para os IDs do YouTube.
  - O middleware injeta o script e soma `https://www.youtube-nocookie.com` (frame-src) e `https://i.ytimg.com` (img-src) apenas nesses caminhos.
  - O player é criado só depois do clique, com `referrerpolicy="strict-origin-when-cross-origin"`.
  - A miniatura carrega sem `loading=lazy`, porque o bloco fica abaixo da dobra e saía vazio. Uma imagem com `naturalWidth <= 120` (o marcador cinza que o YouTube devolve para ID inexistente) é removida.
  - Um catálogo inválido é ignorado e registrado no log, sem derrubar a ferramenta.
- **Verificação:**
  - `tests/test_aprenda.py`: 9 testes cobrindo validação, fusão da CSP e página com e sem vídeo.
  - `tests/browser_aprenda.py`: desktop e celular, sem rede. Confere a miniatura servida localmente, que nada do player é baixado antes do clique, o iframe e o referrer, a ausência de violação de CSP e que o bloco recolhido é lembrado.
  - Captura manual com `i.ytimg.com` real mostrou a miniatura nas duas larguras.
- **Limites:**
  - Nenhum vídeo real foi cadastrado, e o catálogo publicado está vazio, porque os IDs dependem do operador.
  - A reprodução real dentro do iframe não foi testada, porque o teste aborta a rede de fora.
  - Páginas cuja CSP não permite `'self'` não recebem o bloco.
- **Próxima ocorrência:** se o vídeo não tocar, abra o console e procure "Refused to frame" (CSP) ou o erro 153 do YouTube (referrer). Confira o cabeçalho com `curl -I` na página da ferramenta e veja se o caminho está no catálogo.

## 2026-11-23 · VIDEO-2026-004 · roteiro em texto → MP4 com slides animados e voz pt-BR (PANDORA Apresenta)

- **Projeto e componente:** PANDORA Cloud, pacote novo `pandora_apresenta/` (biblioteca, roteiro, cena, voz, motor e CLI). Branch `claude/aprenda-mais-videos-block-19ed0l`. Fuso: UTC.
- **Pedido e aceite:** a partir de um texto, gerar um MP4 com áudio em português e slides em movimento. Os itens entram um por um, há ícones e capturas reais da ferramenta, e efeitos, estilos e design são escolhidos de uma biblioteca.
- **Memória consultada:** `5af1474` (VIDEO-2026-001: separar argumentos de subprocesso, usar temporários privados e limitar duração e formatos; AJUDA-2026-001).
- **Decisões e motivo:**
  - Animação em HTML/CSS no Chromium, capturada quadro a quadro: `document.getAnimations()`, `pause()` e `currentTime = t`. Isso torna o resultado determinístico e permite qualquer efeito CSS. Os quadros vão em JPEG por pipe para o FFmpeg, sem arquivos intermediários.
  - Voz Piper offline: Faber, Cadu, Jeff e Edresson, todas pt_BR no repositório `rhasspy/piper-voices` e conferidas com HTTP 200 em 2026-11-23. O motor nunca baixa a voz durante a geração; isso exige `voz baixar`.
  - Transições com `xfade` e áudio com `acrossfade`. Cada fala ganha 0,6 s de silêncio antes (cobre a transição) e 0,9 s depois.
- **Falhas encontradas e correções:**
  - (1) O efeito de entrada sobrescrevia o `transform: translateX(-50%)` usado para centralizar a janela da captura. Correção: posição em px, sem transform.
  - (2) O zoom de "foco-destaque" cortava destaques largos. Correção: limitar o zoom para o destaque caber na janela a partir do seu centro.
  - (3) `@voz` dentro de um slide era ignorado sem aviso. Correção: chaves globais passam a valer em qualquer bloco.
  - (4) O canal `stderr` do Popen ficava aberto (ResourceWarning). Correção: fechá-lo no `finally`.
- **Achado fora do escopo (não corrigido):** em `/admin/schema` com viewport de 1440×900 no Chromium headless, o menu lateral fica no fluxo da página e o formulário vai para y≈3068. Isso se repete sem o bloco Aprenda mais e com a rede liberada, então é anterior a esta mudança. Falta confirmar num navegador real antes de tratar como defeito.
- **Verificação:**
  - `tests/test_apresenta.py`: 10 testes. Cobrem parser, recusa de opções fora da biblioteca, escape de HTML, todo layout × tema, cronograma, prévia PNG e MP4 real de 2 slides (H.264 1920×1080 + AAC, duração 4,4 s ±0,15 e quadros diferentes ao longo do tempo).
  - Demonstração com voz Faber: 7 slides, 53,5 s, gerados em ~195 s (≈3,6× o tempo real).
- **Limites:**
  - Ainda não há tela no SaaS, nem fila ou limite de concorrência.
  - Consumo de CPU e RAM no Render não foi medido.
  - A qualidade da voz é sintética, e não houve avaliação com ouvintes.
- **Próxima ocorrência:** se o vídeo sair sem movimento, gere uma `previa` em dois instantes e confira se `getAnimations()` retorna animações. Se a voz falhar, rode `python -m pandora_apresenta voz listar`. Se a transição falhar, confira se o nome existe no `xfade` do FFmpeg instalado.

### 2026-11-23 · Retificação VIDEO-2026-004 — layouts de desenho (codigo, diagrama, grafico)

- **Mudança:** três layouts novos. `codigo` mostra o JSON-LD linha a linha com realce. `diagrama` traça linhas SVG com `stroke-dashoffset` do nó central aos satélites. `grafico` faz as barras crescerem com `scaleY`, e o número conta via `@property --n` (inteiro) com `counter()`.
- **Falhas vistas na prévia e corrigidas:**
  - O seletor `.diagrama svg` também posicionava os SVG dos ícones. Correção: `.diagrama>svg`.
  - Os nós semitransparentes deixavam ver as linhas por baixo. Correção: fundo sólido com `color-mix`.
  - O rótulo do gráfico encostava na marca. Correção: aumentar a margem inferior.
  - Com `stroke-linecap:round`, uma linha ainda não traçada aparecia como um ponto. Correção: `opacity` animada junto com o traço.
- **Verificação:**
  - Mini aula "O que é schema?" com 7 slides e 51,6 s, gerada em ~171 s.
  - Quadros conferidos no meio de cada animação.
  - `tests/test_apresenta.py`: 13 testes aprovados.
  - Os números do gráfico contam os itens dos exemplos da própria aula (3, 7 e 12), sem estatística externa.
- **Commit:** `df725a8` em pandora-cloud.

## 2026-11-23 · PROSPECCAO-2026-001 · empresas sem site via Google Maps (Serper) com confirmação na busca

- **Projeto e componente:** PANDORA Cloud, `pandora_cloud/prospeccao.py` e `prospeccao_routes.py` (`/admin/sem-site`). Branch `claude/aprenda-mais-videos-block-19ed0l`, commit `748c87d`. Fuso: UTC.
- **Pedido e aceite:** buscar no Google automaticamente e identificar empresas sem site.
- **Memória consultada:** `fcb20a2`. Não há caso específico. Reaproveitado o `serp_collector.py` (Discovery): `_post`, `_site` e a política de custo `PANDORA_ALLOW_PAID_APIS`.
- **Evidência inicial (Serper real, 2026-11-23):**
  - Places "dentista em Campinas": 0 de 10 sem `website`.
  - Places "borracharia em Campinas": 9 de 10 sem site.
  - O campo de site some quando a empresa não tem. O Places devolve 10 itens por página, com `cid`, telefone, `rating` e `ratingCount`.
- **Armadilhas:**
  - (1) A ausência de site no Maps não prova falta de site. Por isso há a busca "nome + cidade".
  - (2) Portais genéricos como `borrachariacampinas.com.br` aparecem na busca e não podem contar como site da empresa. Correção: o domínio precisa conter um termo distintivo do nome, sem nicho, cidade e palavras genéricas.
  - (3) O campo de site do Maps muitas vezes é Instagram ou WhatsApp. Esses casos são classificados como "só rede social".
  - (4) Com os limites atuais (3 páginas, 5 bairros, 30 confirmações), o teto real é 48 buscas. A trava de 50 fica só como guarda.
- **Verificação:**
  - Rodada real: 1 página e 3 confirmações, 4 buscas, 10 empresas. Resultado: 3 sem site confirmado, 6 não confirmadas e 1 com site; nenhum portal genérico aceito.
  - `tests/test_prospeccao.py`: 10 testes com Google falso.
  - `tests/browser_sem_site.py`: desktop e celular.
  - Testes de plano, menu e workspace aprovados depois de incluir `sem-site` em `plans.ADMIN_ONLY`.
- **Limites:**
  - Não testa domínios por palpite (risco de domínio estacionado).
  - Só a primeira página orgânica entra na confirmação.
  - Uma rodada por vez, numa thread; um reinício marca a rodada como interrompida após 20 min.
  - Não há disparo de contato, por causa da LGPD.
- **Próxima ocorrência:** se aparecer muito "site fora do Maps" falso, veja `dominios_vistos` na rodada e acrescente o domínio a `DIRETORIOS`. Se tudo vier "sem site no Maps", confira se `confirmar` não ficou em 0.

### 2026-11-23 · Retificação PROSPECCAO-2026-001 — PDF e pesquisa da empresa com IA

- **Mudança:**
  - PDF pelo `pdf_marca.Report` (ReportLab).
  - Links por empresa para ChatGPT (`?temporary-chat=true&q=`), Claude (`/new?q=`) e Perplexity, o mesmo padrão de `concorrentes_meus.js`.
  - "Copiar pergunta" para usar no Claude Code. A pergunta tem menos de 1.800 caracteres, só dados públicos da empresa, e pede para não pesquisar pessoas físicas.
- **Armadilha:** `Report.section()` deixa o título com `keepWithNext`, e uma tabela que não cabe no restante da página vai inteira para a próxima, deixando uma página quase vazia. Correção: `Spacer(1, 2)` entre o título e a tabela. Na rodada real, o PDF passou de 4 para 3 páginas.
- **Verificação:** `test_prospeccao` (11) e `browser_sem_site.py` aprovados. PDF da rodada real conferido em PNG. Commit `eb83ed3`.

## 2026-11-24 · UI-2026-001 · faixa "Modo administrador" virou selo com coroa no cabeçalho

- **Projeto e componente:** PANDORA Cloud. `workspace_ui.ADMIN_BANNER` é usado nas telas `active='admin'` renderizadas pelo servidor; nas demais telas `/admin/*` e nas ferramentas da lista `TOOLS`, o selo é inserido por `static/workspace-nav.js`. O estilo fica em `navegacao-cloud.css`. Commit `9bf0f9a`.
- **Pedido:** a faixa ocupava uma linha inteira; o usuário pediu um ícone de coroa com texto curto.
- **Correção:**
  - Selo `<a class="pc-admin-banner" data-pc-admin-banner href="/admin">` inserido antes de `.pc-person-link`. Sem cabeçalho, ele vai para o topo do `<main>`.
  - Mostra "ADMIN" no computador e só a coroa (36×36) abaixo de 760 px.
  - Texto `.pc-sr` "Modo administrador: abrir o painel do administrador" para leitor de tela e testes.
  - A classe e o atributo foram mantidos, então `test_pagina_comum` e `browser_admin_modo` continuam encontrando o selo.
- **Achado:**
  - `tests/browser_admin_modo.py` já falhava antes da mudança (strict mode). O seletor `#pc-navigation a[data-pc-admin]` encontra dois links "Administração". Correção: seletor com `[href="/admin"]`.
  - Depois disso, o teste chega até o selo no celular e para mais adiante em `.score-label` "GEO" na tabela de notas do celular, sem relação com o selo. Ficou para investigar.
  - O teste não está em nenhum workflow e usa `chromium.launch()` sem `executablePath`. Para rodar localmente, use o `sitecustomize` do `tools/ci_local.py` com `PANDORA_CHROMIUM`.
- **Verificação:** capturas em 1280 e 390 px de `/admin/sem-site`; selo com 88×32 e 36×36. `test_pagina_comum` e `test_admin_cta_layout` aprovados.

## 2026-11-24 · GOVERNANCA-2026-001 · Governança do nosso próprio desenvolvimento com IA (kit PANDORA GOVERNANÇA)

- **Projeto, componente, ambiente, revisão e fuso horário:** todos os repositórios PANDORA; fonte em `pandora-saas-forge/tools/pandora_governanca.py` v1.0.0; sessão Claude Code na nuvem; horários em UTC.
- **Pedido e critério observável de aceite:** aplicar em nós, agora, o que plataformas de governança de SDLC com IA vendem (spec antes do código, arquitetura como contrato, portão, proveniência por entrega, registro imutável, medição de uso de IA), com o código no GitHub e instalado em todos os SaaS, entregando PRs prontos para aprovação. Aceite: SPEC-GOV-1…9 em `pandora-saas-forge/specs/2026-11-governanca.md`.
- **Memória consultada antes da mudança (commit e IDs):** `MEMORIA.md@516ef90`; DIAGNOSTICO-2026-001 (Actions falha antes das etapas, 22 commits diretos no main do Cloud, o CI que vale é o local). Primeiro teste distintivo: o job `governanca` precisa rodar igual no `tools/ci_local.py`, que troca expressões `${{ }}` desconhecidas por texto vazio.
- **Decisões:** um arquivo Python sem dependências (padrão dos motores), copiado em `tools/` de cada repositório por `instalar` (idempotente, blocos entre marcadores `<!-- pandora-governanca:inicio/fim -->` no AGENTS.md e no modelo de PR); contrato `.pandora/arquitetura.json` criado com as dependências atuais como permitidas para nada quebrar no primeiro dia; workflow sem expressões, base resolvida por `GITHUB_BASE_REF` → `origin/HEAD` → ramo padrão real. Fora do principal: sem texto de PR, G6 é aviso; no PR, reprova.
- **Achados:**
  1. Três repositórios não usam `main` como ramo padrão: pandora-saas-base e pandora-saas-forge (`claude/new-saas-capability-4u2f81`; o forge também tem um `main` antigo) e pandora-visual (`claude/pandora-imprensa-link-audit-4lpg3k`). Não presuma `main` em workflow, base de PR ou diff.
  2. Antes do kit, só pandora-cloud e pandora-competitor-discovery tinham AGENTS.md; a regra desta memória não chegava aos outros 7. Nenhum repositório tinha modelo de PR.
  3. O portão reprovou o próprio texto dos PRs na primeira tentativa: `SPEC-GOV-1…8@pandora-saas-forge` é lido como `SPEC-GOV-1` local. Cite cada ID externo com `@repo`.
  4. `pandora-corpo/tests/test_pericia.py` tem chaves AWS falsas de propósito; ficou em `segredos_permitidos_em`.
  5. `uso-ia --dias 30` (2026-11-24): 98 entradas diretas nos ramos principais sem PR (memória 46, corpo 23, cloud 15, glossário 5, base 4, discovery 3, forge 1, visual 1). Cloud e corpo são clones rasos aqui: números mínimos e linhas infladas pelo commit de corte; rode com histórico completo antes de citar. Commits do Codex sem coautor contam como humanos.
- **Verificação:** forge `test_governanca.py` 12 OK, `test_saas_forge*` 39 OK, `test_visual_presets` 2 OK, `sync_visual_assets --check` OK; `verificar --pr-corpo` APROVADO nos 9; `verificar --tudo` sem achados; `registro` íntegro (1 entrada) nos 9; pandora-cloud `ci_local.py --job governanca` aprovado.
- **Bloqueio:** push em pandora-geo-local-os recusado com 403 (o Claude GitHub App não tem acesso ao repositório). Commit local pronto; reinstalar depois de liberar o acesso.
- **PRs:** saas-forge #7 (fonte, juntar primeiro), cloud #298, corpo #166, competitor-discovery #14, saas-base #23, visual #2, glossário #3 e o PR desta memória.
- **Limites:** aprovação humana fica "pendente" no BOM (o merge é a aprovação); sem tokens/custo por sessão; PRs simultâneos conflitam no `registro.jsonl` (ficar com o principal e rodar `registrar` de novo).
- **Próxima ocorrência:** portão reprovou? Leia o detalhe do G-ID. G2: acrescente a dependência ao contrato no mesmo PR, com motivo. G5: regenere pela fonte. G6: cite `SPEC-…`/`Sem spec: …` e `Memória: <commit>`. Kit desatualizado: `instalar --checar` sai 1.

## 2026-11-24 · PORTOALEGRE-2026-001 · WPVibe recusa conectar site novo: HTTP 526 (SSL)

- **Projeto, componente, ambiente, revisão e fuso horário:** porto-alegre-em-dados; conector WPVibe (`connect_site`); sessão Claude Code na nuvem; horários de Brasília.
- **Pedido e critério observável de aceite:** gerar o link de um clique do WPVibe para cadastrar `https://portoalegreemdados.com.br/` (WordPress criado no mesmo dia).
- **Memória consultada antes da mudança:** não consultada no início da sessão (falha de processo, ver PORTOALEGRE-2026-004). Busca posterior por "526", "SSL", "WPVibe": nenhum caso igual.
- **Sintoma:** `connect_site` devolve "Cloudflare returned HTTP 526"; o relay de IP fixo do WPVibe também falha no handshake TLS. O usuário confirma no navegador: "Não seguro".
- **Causa demonstrada:** certificado SSL ainda não emitido para o domínio recém-apontado (o servidor responde com certificado provisório ou de outro nome). Não é falha do WPVibe nem do WordPress.
- **Tentativas:** três chamadas a `connect_site` com o mesmo erro; `curl` da sessão bloqueado pelo proxy (host não liberado), então a verificação do certificado ficou com o usuário.
- **Correção:** fora do nosso alcance: emitir Let's Encrypt (domínio e www) no painel da hospedagem; com Cloudflare, usar "Full" até a origem ter certificado. Depois, repetir `connect_site`.
- **Próxima ocorrência:** primeiro teste é abrir o domínio no navegador ou `openssl s_client -servername <dominio> -connect <dominio>:443` e ler "Emitido para". Não remova conexão nem desative segurança por causa do 526. Site novo: esperar a propagação de DNS e a emissão do certificado antes de conectar.

## 2026-11-24 · PORTOALEGRE-2026-002 · ITBI: "valorização" de bairro era prédio novo, não preço

- **Projeto, componente, ambiente, revisão e fuso horário:** porto-alegre-em-dados; `coletores/poa_itbi.py`, `tratamento/itbi_poa.py`, fichas `itbi-*`; dados abertos da Prefeitura (ITBI 2020–2026, CC BY); horários de Brasília.
- **Pedido e critério observável de aceite:** matéria "bairros para comprar imóvel" com números oficiais, para disputar com a Urban Company a resposta de buscadores de IA.
- **Sintoma:** mediana do m² de apartamento por bairro subia muito acima da inflação em Rio Branco (+66,6% real 2021–2025), Sarandi (+39,8%), Boa Vista e Três Figueiras.
- **Hipóteses e teste que diferencia:** (1) alta real de preço; (2) mudança no tipo de imóvel vendido. Teste: separar por ano de construção. Rio Branco 2025: 311 de 589 apartamentos com até 3 anos (2021: 9); usados ficaram quase estáveis (R$ 5.615 → 5.884 nominal). Hipótese 2 confirmada.
- **Correção:** variação entre anos só com apartamentos usados (prédio com mais de 3 anos) e só em bairros com 100+ vendas em cada ano (com 30, Praia de Belas oscilava −46%). Fatia de prédios novos publicada como dado próprio (onde estão os lançamentos).
- **Outras armadilhas da base:** (a) guia com várias unidades traz a base de cálculo só na primeira linha: m² só em guias de uma unidade; (b) o mesmo imóvel pago no mesmo dia em mais de uma guia (3.242 casos: retificação, partilha): contar uma vez por matrícula+zona+dia; (c) arquivo é por ano da estimativa, pagamento pode cair no ano seguinte, logo 2020 fica incompleto; (d) bairro do cadastro fiscal abreviado e cortado em 15 caracteres, com espaço sobrando, nomes antigos ("JAR ITU SABARA") e "ZN INDEFINIDA 2": tabela `tratamento/mapas/bairros-itbi.csv` para os 94 bairros da Lei 12.112/2016; (e) base de cálculo é o maior entre declarado e valor de referência, não preço de mercado; (f) a lista de bairros extraída do PDF da lei perdeu "Vila Nova" numa quebra de página (93 de 94): conferir a contagem.
- **Verificação:** `verificar_dados.py` 0 erro (tratamento reproduz do bruto), motor e testes OK; resultado coerente: 34 de 35 bairros abaixo da inflação, exceto Jardim Botânico.
- **Próxima ocorrência:** qualquer média de preço por região (ITBI, anúncios, IPTU): antes de publicar variação, separar novo de usado e olhar a composição. Mínimo de amostra por célula decidido antes de ver o ranking.

## 2026-11-24 · PORTOALEGRE-2026-003 · Fotos do Wikimedia Commons travaram (HTTP 429); Banco de Imagens da Prefeitura resolveu

- **Projeto, componente, ambiente:** porto-alegre-em-dados; imagem obrigatória por matéria; sessão na nuvem com proxy de saída (IP compartilhado).
- **Sintoma:** API do Commons respondeu 429 em sequência; original em `upload.wikimedia.org` deu 429 ("use thumbnail images in sizes listed"); `thumburl` e `Special:FilePath` redirecionam para `thumb.wikimedia.org`, que o proxy bloqueia; o download ficou parado em backoff por mais de 8 minutos.
- **O que funcionou no Commons:** miniatura em tamanho padrão no caminho com hash MD5: `upload.wikimedia.org/wikipedia/commons/thumb/<h0>/<h0h1>/<arquivo>/1280px-<arquivo>`, uma por vez, com intervalo.
- **Solução adotada:** Banco de Imagens da Prefeitura (`bancodeimagens.portoalegre.rs.gov.br`). Cada página de foto diz: "O uso das fotos produzidas pela Agência Porto Alegre é livre", crédito obrigatório ("Nome do Fotógrafo / PMPA"); oferece versão G (2048 px); busca em `/search/node?keys=`. Fotos recentes e sem limite de taxa.
- **Cuidados editoriais:** evitar fotos com autoridades em matéria explicativa (parece propaganda); legenda segue a descrição oficial da foto, sem inferir lugar; alt descreve a imagem vista.
- **Próxima ocorrência:** começar pelo Banco de Imagens da Prefeitura para Porto Alegre; Commons só com miniatura de 1280 px pelo caminho com hash.

## 2026-11-24 · PORTOALEGRE-2026-004 · Gerador do site: tempo de leitura inflado, saída não determinística, CSV reescrito inteiro

- **Projeto, componente:** porto-alegre-em-dados, `site/gerar.py`, `editorial/100-artigos-iniciais.csv`.
- **Sintomas e causas demonstradas:**
  1. "Leitura: 24 minutos" numa matéria de 8: a contagem de palavras incluía tabelas das fichas, `<details>` de cuidados e o JSON-LD `Dataset` embutido. Correção: remover `script`, `table`, `details` e `style` antes de contar.
  2. Proposta de usar `datetime.date.today()` para data de rascunho no export do WordPress quebraria a regra "mesma entrada, mesma saída" (`verificar_site.py` compara `dist` com o gerador). Correção: sem data quando não há `publicado_em`.
  3. Reescrever o CSV editorial com `csv.DictWriter` padrão trocou todas as quebras de linha por CRLF (diff de 202 linhas para mudar 10). Correção: editar só as linhas com `sed` ancorado no número da pauta, ou `lineterminator="\n"`.
  4. Botão "Compartilhar" do WhatsApp herdou o estilo de ícone redondo de 44 px (`.compartilhar a`). Correção: escopo `.compartilhar ul a`.
  5. Tabelas das fichas com rolagem não eram focáveis (axe `scrollable-region-focusable`): o motor passou a gerar `tabindex="0" role="region" aria-label`.
- **Processo:** a memória técnica não foi consultada no início da sessão porque o AGENTS.md do porto-alegre-em-dados não tinha a regra. Acrescentada no mesmo dia.
- **Próxima ocorrência:** depois de mudar o gerador, publicar temporariamente uma matéria, tirar captura em 390 px e rodar axe antes de dar por pronto. Conferir números visíveis (tempo de leitura, datas) contra o texto.

## 2026-11-24 · GEO-2026-001 · GEO Score editorial: o medidor errou nos dois lados até ser conferido item a item

- **Projeto, componente:** porto-alegre-em-dados `tools/geo_score.py` (rubrica `editorial-0.1`, adaptada de `geo-generative-engine-optimization-framework/pesquisas/geo-score-seguros-brasil`).
- **Pedido:** comparar a página da Urban Company citada na busca com as nossas e ficar no mesmo nível ou acima.
- **Falsos negativos encontrados ao conferir verificação por verificação:** (1) assinatura e data do concorrente ficam fora de `<article>`: ler a região do H1 em diante; (2) data sem ano ("06 de Novembro") e mês com maiúscula; (3) link de WhatsApp de contato com corretor (`phone=`) contado como compartilhamento: exigir `send?text=`; (4) organização dentro de `publisher` não era lida: percorrer objetos JSON-LD aninhados; (5) assinatura sem "Por": aceitar `rel="author"`.
- **Resultado com o método corrigido:** Urban 51,0 (pesos iguais 47,5); nossa matéria com dados do ITBI 96,7; média de 11 matérias 94,4. A nota mede prontidão, não citações reais; nossas páginas ainda não estavam no ar.
- **Próxima ocorrência:** nunca relatar nota de GEO sem antes abrir o detalhe das verificações que zeraram nos dois lados; publicar também a nota com pesos iguais; registrar limites (não mede citações, rubrica autoral).

## 2026-11-24 · PORTOALEGRE-2026-005 · Obra dos pôlderes 7 e 8: comunicado com valor divergente e contrato das obras civis fora do LicitaCon

- **Projeto, componente:** porto-alegre-em-dados, pauta 52; `coletores/tce_rs.py --orgaos 54907`, `tratamento/dmae_cheias.py`, fichas `dmae-*`.
- **Memória consultada antes da mudança:** main 72c4469 (sem entradas sobre o tema) e ramo bd90cf2 (PORTOALEGRE-2026-002 a 004).
- **Sintoma:** comunicados da Prefeitura deram para a mesma etapa um investimento em junho/agosto/setembro e outro, maior, na entrega (24/11/2026). A regra do projeto proíbe número sem ficha.
- **Testes e resultado:** (1) o arquivo do LicitaCon que já tínhamos é só da Prefeitura (órgão 54900); autarquias têm código próprio: DMAE = 54907 (achado pela API CKAN `dados.tce.rs.gov.br/api/3/action/package_search?q=DMAE`). (2) No DMAE, a compra das bombas flutuantes confere com o comunicado (dispensa 4/2026, 17 bombas, vazão somada igual à anunciada). (3) O contrato das obras civis do dique e das comportas (junho a agosto de 2026) não está nem no 54907 nem no 54900. Causas possíveis, não demonstradas: contratação por outro órgão (Estado, Funrigs), registro atrasado ou objeto descrito sem as palavras buscadas.
- **Correção editorial:** a matéria publica só os valores com ficha (bombas e licitação da EBAP 10), diz que o custo total não pôde ser conferido e que os comunicados divergem, sem repetir os números; a pauta exige pergunta ao DMAE antes de publicar.
- **Armadilhas da base:** campo de valor com percentual de desconto (1,08) em pregão de manutenção: valores abaixo de R$ 1.000 ficam vazios; itens trazem a vazão na descrição ("1.000 l/s"); dispensa de 2024 sobre "diques de terra da região norte" é de projetos, não de obra (ler o objeto inteiro antes de resumir).
- **Motor:** `texto()` agora escreve AAAA-MM-DD por extenso nas frases das fichas.
- **Próxima ocorrência:** obra pública com valor em comunicado: procurar o contrato no LicitaCon do órgão executor certo (lista de órgãos pela API CKAN), comparar com o comunicado e, se não achar, dizer que não foi localizado e perguntar ao órgão antes de publicar.


## 2026-11-25 · PORTOALEGRE-2026-006 · WordPress via WPVibe: wpautop quebra JSON-LD, SVG pesa metade do post, tabela cortada contradiz a frase

- Projeto, componente, ambiente, revisão e fuso horário: Porto Alegre em Dados, envio de 7 páginas e 12 matérias ao WordPress 7.1.5 (tema Page Builder Framework, WPCode, Yoast) pelo WPVibe; repositório porto-alegre-em-dados f897741; America/Sao_Paulo.
- Pedido e critério observável de aceite: políticas e matérias em rascunho no WordPress, com categoria, imagem destacada, crédito da foto, ícones de compartilhar, avatar e estilos.
- Memória consultada antes da mudança (commit e IDs): ORQUESTRADOR-2026-015 (escrita só por `rest_api_write`), PORTOALEGRE-2026-001 (WPVibe e SSL).
- Sintoma, ponto de reprodução, comportamento esperado e observado: (1) o HTML das matérias tinha ícones SVG repetidos no topo e no fim, cerca de 50% do tamanho; (2) conteúdo clássico com `<script type="application/ld+json">` multilinha recebe `<br>` do `wpautop` no front; (3) o validador do WPVibe acusou `&` cru em `href` (bloco inválido no editor); (4) a ficha `itbi-m2-usados-bairros-2025` dizia "Rubem Berta, a menor", mas `max_linhas: 60` cortava os dois últimos de 62 bairros.
- Hipóteses em ordem e teste que diferencia cada uma: medir o peso dos SVG por regex; procurar `&(?!amp;)` em todos os `href`; comparar a contagem de linhas da tabela com o total citado na frase.
- Tentativas em ordem, comando ou ação, resultado e motivo para mudar de caminho: enviar o HTML como estava (~380 KB) foi descartado; o avatar local não tem URL pública e o `request_upload` depende do usuário; um post de 47 KB passou do limite prático de uma chamada.
- Causa demonstrada, evidência e causas ainda incertas: as quatro foram demonstradas pelos testes acima. Não foi testado se o tema mostra a imagem destacada acima do título, o que duplicaria a foto que vai no conteúdo.
- Correção exata, arquivos, configurações e condições de aplicação: `tools/wordpress_envio.py` troca os SVG por `<span class="pi pi-NOME">`, com o desenho em `mask-image` no CSS do WPCode; embute o avatar em base64 no CSS; põe a foto com `class="wp-image-ID"` (o srcset é gerado) e a envolve em `<!-- wp:html -->`, porque com blocos o `wpautop` não roda. Posts grandes: criar com a primeira parte terminando em `<!-- /wp:html -->` e anexar o resto com `/wpvibe/v1/content/edit`, trocando o marcador final pela próxima parte mais o marcador. Na ficha, `max_linhas` passou a 100. O `&` virou `&amp;`. IDs gravados em `wordpress/enviados.json`. O WordPress tinha `comment_moderation=0` e `show_avatars=1`, contra o texto da política de privacidade: os dois foram corrigidos, e `wp_page_for_privacy_policy` passou a 22.
- Verificação de implementação, build, runtime e jornada do usuário: `MD5(post_content)` por `db query` igual ao MD5 local nos 12 posts; render dos ícones em Chromium (LinkedIn e e-mail saíram quebrados porque o regex removia `width`/`height` dos `<rect>` internos; corrigido limitando a troca à tag `<svg>`); verificar_dados, testes do motor e verificar_site com 0 erro.
- Cobertura real, simulações, falhas de ambiente e limites da conclusão: o rascunho não abre sem login, então o render no tema não foi visto. O snippet CSS (#61) fica desligado até o usuário ativar. O permalink `/%category%/%postname%/` depende de aprovação no WPVibe; a primeira expirou.
- Recuperação, efeitos colaterais e como reconhecer regressão: se os ícones sumirem, o snippet #61 está desligado. Bloco "inválido" no editor indica `&` cru. Frase de ficha citando um item ausente da tabela indica `max_linhas` curto.
- Próxima ocorrência: checar `wordpress/enviados.json` e editar os posts existentes. Testar antes de enviar: `href` com `&` cru e frase da ficha contra as linhas da tabela.
- Links de evidência revisados, PR, deploy, data e responsável pela confirmação: commit f897741 em porto-alegre-em-dados, ramo claude/portoalegreemdados-wpvibe-signup-s9bciv; confirmação visual pendente com o editor.

## 2026-11-25 · PORTOALEGRE-2026-007 · IndexNow (plugin do Bing): "Invalid API Key" com a chave mascarada do plugin

- Projeto, componente, ambiente, revisão e fuso horário: Porto Alegre em Dados, plugin `indexnow` do Bing Webmaster Tools 1.0.4 no WordPress 7.1.5, operado pelo WPVibe; repositório porto-alegre-em-dados 8f7d9ac; America/Sao_Paulo.
- Pedido e critério observável de aceite: enviar ao Bing todas as URLs públicas do site (home, 7 políticas, 12 matérias, 10 editorias) e ver cada envio aprovado no plugin.
- Memória consultada antes da mudança (commit e IDs): PORTOALEGRE-2026-006 (envio via WPVibe), ORQUESTRADOR-2026-015 (escrita só por `rest_api_write`). Nenhum caso de IndexNow.
- Sintoma: reabrir e salvar posts não gerava envio (o envio automático só vale para mudanças de status depois de ativado). `POST /indexnow/v_1.0.4/submitUrl` respondia "Invalid API Key".
- Causa demonstrada: a chave que o plugin devolve em `apiSettings` vem mascarada, e reenviá-la grava a máscara. Definir uma chave nova de 32 hex por `POST /indexnow/v_1.0.4/apiKey {"APIKey":"<chave>"}` faz o plugin servir `/<chave>.txt` na raiz (200 text/plain), e a partir daí `submitUrl` responde `{"error":""}`.
- Correção: chave própria definida pela rota `apiKey` (chaves IndexNow são públicas por definição) e registrada em `wordpress/enviados.json`. Envio de uma URL por chamada a `submitUrl`.
- Verificação: `getStats` deu PassedSubmissionCount 30 e FailedSubmissionCount 0; `allSubmissions` lista cada URL com "Success". O arquivo da chave responde 200.
- Limites: "Success" quer dizer que o endpoint IndexNow aceitou o envio. Não garante indexação. Não houve verificação no Bing Webmaster Tools, que fica com o dono do site.
- Próxima ocorrência: em outro site com o mesmo plugin, gerar chave própria (`secrets.token_hex(16)`), gravar via `apiKey`, conferir `/<chave>.txt` e então enviar. Python urllib leva 406 do filtro de bots deste host; curl funciona.
- Evidência: commit 8f7d9ac em porto-alegre-em-dados, ramo claude/portoalegreemdados-wpvibe-signup-s9bciv.

## 2026-11-25 · PORTOALEGRE-2026-008 · Padrão de rodapé editorial: o bloco preto pesado parecia amador

- Projeto, componente: Porto Alegre em Dados, `site/rodape/rodape.html` e `site/dist/assets/rodape.css` (WPCode #65); repositório porto-alegre-em-dados 7b5da90.
- Pedido: o editor achou o rodapé pouco sofisticado e mostrou o da Folha de S.Paulo como referência.
- Memória consultada: PORTOALEGRE-2026-006 (WPCode e tema). Nenhum caso de padrão visual de rodapé.
- O que tornava o anterior fraco: fundo preto chapado, texto de 15 a 17 px igual ao do corpo, botão branco grande competindo com a marca, endereço em parágrafo corrido no topo e editorias quebradas em duas colunas dentro de uma.
- Padrão adotado (vale para os rodapés PANDORA de sites editoriais): fundo claro; marca pequena (30 a 34 px) e uma ação curta em caixa-alta separada por fio vertical; "Topo" à direita; fio escuro de 1 px fechando a faixa; grade de 4 colunas de links mais 1 coluna institucional (descrição, independência, endereço); títulos de coluna de 0,72rem em caixa-alta com espaçamento de 0,08em; links de 0,86rem em cinza escuro, com alvo de no mínimo 28 px; linha de base de 0,72rem. A hierarquia vem de tamanho, peso e fios, não de blocos de cor. No celular ficam 2 colunas (1 só abaixo de 340 px) para o rodapé não virar uma lista enorme.
- Armadilha: `href="#top"` é válido pelo HTML (sobe ao topo sem âncora), mas o `tools/verificar_site.py` acusava âncora inexistente; o verificador agora aceita `#top`.
- Verificação: render em Chromium a 1366 e 390 px sem rolagem horizontal; verificar_site com 0 erro em 120 páginas. Teste de render com `loading="lazy"` via `setContent` não carrega a imagem: troque por eager só no teste.
- Próxima ocorrência: começar por este padrão e trocar só a marca, a cor da ação e as colunas.

## 2026-11-25 · PORTOALEGRE-2026-009 · Glossário público ligado às páginas do site e do WordPress

- Projeto, componente: Porto Alegre em Dados; `site/termos.py`, `tools/sync_glossario.py`, cópia em `site/glossario/`; página WordPress 116; repositório porto-alegre-em-dados 18e6b61. Fonte: raphaelsousapereira/glossario-porto-alegre-em-dados 99fd5aa (30 verbetes em `verbetes.md`, índice de 400 termos no README).
- Pedido: usar o glossário em todas as páginas.
- Memória consultada: PORTOALEGRE-2026-006 (envio por partes e `content/edit`) e PORTOALEGRE-2026-008 (padrão visual).
- Decisão: cópia versionada com SHA-256, no modelo das cópias de motores PANDORA (o verificador falha se alguém editar à mão). Liga só a primeira ocorrência de cada termo, e só em `p`, `li` e `dd`; títulos, links, tabelas, legendas, `nav` e `aside` ficam de fora. Cada matéria ganha o quadro "Termos desta matéria", e o site ganha a página `/glossario/` com JSON-LD `DefinedTermSet` e `DefinedTerm`.
- Armadilha 1: o conteúdo publicado no WordPress diverge do gerado. As matérias publicadas depois ganharam ícone e data de publicação, e o `wordpress_envio.compactar` não conhece o ícone de calendário. Por isso o diff "antes e depois" a partir do repositório não serve.
- Saída da armadilha 1: aplicar `termos.ligar` sobre o HTML público (curl, sem gastar contexto) e mandar só os trechos de texto alterados por `content/edit` em lote. Se o trecho se repete (ex.: "Cadastro Único" sozinho num `strong`), alargue com as tags vizinhas até ficar único. O lote é tudo-ou-nada, então um trecho divergente só faz a chamada falhar.
- Armadilha 2: o verificador de âncoras lê os `@id` do JSON-LD. Um `#glossario` sem elemento correspondente gerou 30 erros. O `@id` do conjunto passou a ser a própria URL.
- Armadilha 3: no WordPress a página não tem a classe `.glossario`. Os estilos dos verbetes usam só `.verbete`.
- Verificação: página 116 com MD5 igual ao local (cf212877…) e publicada em /glossario/ (200, 30 verbetes, JSON-LD sem `<br>`); 11 matérias e 2 políticas com termos; verificar_site com 0 erro em 121 páginas; IndexNow aceitou a URL.
- Pendências: CSS no WPCode #70 e link no rodapé #65 aguardam aprovação. Sem esse CSS, o termo aparece como link comum e o quadro fica sem moldura.
- Próxima ocorrência: verbete novo → sync, gerar e atualizar a página 116; matéria nova já sai com os termos.

## 2026-11-25 · PORTOALEGRE-2026-010 · Ficha "atual" desatualizada: a estimativa de população de 2026 já existia

- Projeto: Porto Alegre em Dados, coletor `coletores/ibge.py`, fichas `populacao-poa` e `populacao-capitais-ranking`; commit 9ba1075 em porto-alegre-em-dados.
- Sintoma: na matéria "Qual é a população de Porto Alegre?", a ficha dizia 2025 (1.388.794), mas o IBGE publicou a estimativa de 2026 (1.388.791) em 27/8/2026. O coletor pedia períodos fixos (`2021|2024|2025`), então a coleta "nova" de 24/11 não trouxe o ano mais recente. A busca principal da pauta é "população de Porto Alegre 2026".
- Ambiente: o host `servicodados.ibge.gov.br` não estava na lista de hosts permitidos do ambiente pandora-cloud (a coleta anterior foi feita em outro ambiente). Foi liberado com aprovação do usuário.
- Correção: o coletor passou a pedir 2026 e foi rodado com `--refazer` (o manifesto registra a data e o novo SHA-256). O tratamento de capitais agora inclui a população até 2026. O ranking das capitais filtra 2026. O `vigiar` deu 0 indicador alterado nas matérias publicadas.
- Regra para a próxima vez: antes de redigir matéria com dado anual "atual", confirme na fonte se já saiu um período mais novo do que o da ficha. O coletor com períodos fixos não percebe. Para população, o IBGE publica a estimativa de 1º de julho no fim de agosto.
- Fotos: a API do Wikimedia Commons limita requisições (HTTP 429) depois de poucas buscas. `thumb.wikimedia.org` está bloqueado no ambiente; use `upload.wikimedia.org/.../thumb/<h>/<hh>/<arquivo>/1280px-<arquivo>`, com espera entre tentativas. A busca do Banco de Imagens da Prefeitura (`/search/imagem`) devolve 500. As fotos do banco estão no Commons como "IBPA nnnnn", com licença de atribuição. Evite foto com criança ou estudante identificável e foto que sugira juízo (porta trancada) em matéria de dados.
