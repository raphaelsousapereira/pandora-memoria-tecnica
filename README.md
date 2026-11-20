# PANDORA · Memória técnica de desenvolvimento e operação

O arquivo vivo é [MEMORIA.md](MEMORIA.md). Ele mantém a cronologia de cada investigação, as tentativas, os comandos aplicados, os testes e quem observou o resultado. As revisões ficam no Git. Use o histórico de commits para recuperar a versão consultada durante um incidente.

## Como começar em um projeto

Na raiz do projeto consumidor, acrescente o texto de [INTEGRACAO.md](INTEGRACAO.md) ao AGENTS.md existente. Em CLAUDE.md, acrescente uma linha isolada `@AGENTS.md` se ainda não existir. Preserve as regras de teste, deploy e revisão já presentes nos arquivos. Distribua esse par de entradas no template de novos projetos. Projetos criados anteriormente precisam receber a entrada individualmente. A recuperação da memória central no início de cada trabalho mantém todos consultando a versão atual.

No Codex, inicie a sessão dentro do projeto para carregar as instruções da árvore correspondente. No Claude Code, confira em /memory quais arquivos foram carregados. No ChatGPT comum, forneça o acesso ao repositório e inclua a regra de INTEGRACAO.md nas instruções do Projeto ou na tarefa. Uma conversa sem acesso ao GitHub precisa receber os arquivos como contexto. Ter um arquivo central privado não injeta seu conteúdo em todos os chats.

## Leitura e atualização reproduzíveis

Quem usa terminal pode consultar assim, em um diretório de trabalho separado:

```sh
git clone https://github.com/raphaelsousapereira/pandora-memoria-tecnica.git
cd pandora-memoria-tecnica
git pull --ff-only origin main
git rev-parse HEAD
cat AGENTS.md
rg -in 'microfone|Whisper|Render' MEMORIA.md
less MEMORIA.md
```

Nas próximas sessões reutilize o clone e atualize a main. Com conector, leia os mesmos caminhos na main e use o SHA da revisão da branch para registrar a consulta. No caso de agentes concorrentes, cada um abre uma branch, acrescenta uma ficha com identificador próprio, atualiza o índice e abre PR. Se a base avançou, integrar a alteração preservando as duas entradas. No conector de arquivos, obtenha o blob SHA mais recente antes de gravar; em conflito, releia e reaplique o acréscimo. Não substitua o arquivo inteiro por uma cópia antiga.

## Critério de revisão

O responsável pela revisão confere quatro itens: a revisão da memória consultada, o caso aplicável, a evidência de aceite do produto e o commit que acrescentou o aprendizado (ou justificativa de que o caso já cobria o resultado). Se faltar algum deles, devolver para complemento antes de declarar o trabalho finalizado. Esta obrigação é uma regra operacional verificável por evidências. O Markdown não comprova leitura humana ou por modelo e não configura proteção de branch. Esse fluxo independe de minutos de GitHub Actions.

## Referências de implementação

- [OpenAI · descoberta de AGENTS.md](https://developers.openai.com/codex/guides/agents-md/)
- [Anthropic · memória e imports do Claude Code](https://code.claude.com/docs/en/memory)
- [GitHub · criação a partir de templates](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template)
