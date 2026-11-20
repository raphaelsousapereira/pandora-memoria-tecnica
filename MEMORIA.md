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
