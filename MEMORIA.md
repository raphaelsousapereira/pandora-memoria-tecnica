# Memória técnica acumulativa da PANDORA

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
