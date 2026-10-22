# Integração Claude Code + Codex + Grok

Skill do Claude Code para trabalhar com um **segundo e um terceiro motor de IA** na mesma máquina: o Codex, pelo plugin oficial da OpenAI, e o Grok, pelo CLI. O Claude conduz, os outros dois produzem, e nada vira fato sem ser conferido.

Feito pela [Singular TEC](https://singularhub.com.br) ([@singular_tec](https://www.instagram.com/singular_tec/)). Licença MIT. Em português do Brasil.

## A dor

Você manda outro modelo revisar seu código e recebe um relatório confiante com um achado "crítico". Ou você acredita e refatora à toa, ou desconfia e joga tudo fora. Nos dois casos, a delegação virou teatro.

O ganho de chamar outro motor não é velocidade. É **viés diferente**: um modelo treinado de outro jeito enxerga o que o seu não enxerga. O custo é que ele também inventa com a mesma confiança. Por isso a skill gira em torno de uma regra:

> **Eles produzem, o Claude verifica e assina.** Achado de outro motor é hipótese até ser reproduzido no código.

## O que o plugin oficial faz e o que esta skill acrescenta

Desde março de 2026 a OpenAI mantém o [`codex-plugin-cc`](https://github.com/openai/codex-plugin-cc), que coloca o Codex dentro do Claude Code (`/codex:review`, `/codex:adversarial-review`, `/codex:rescue`, trabalhos em segundo plano). Ele resolve **a chamada**. Para o Grok não existe plugin oficial.

| O plugin oficial resolve | Esta skill acrescenta |
|---|---|
| Chamar o Codex de dentro do Claude Code | Quem faz o quê: Codex ataca e faz correção pequena, Grok levanta e faz o braçal em vários arquivos, Claude decide e verifica |
| Revisão só leitura e adversarial | O contrato de ataque: vetores numerados, contra-advertência do runtime, licença para não achar nada, teto de palavras |
| Trabalho em segundo plano | Como o Claude dispara a revisão sozinho (os comandos de review só podem ser digitados pela pessoa) |
| | O Grok headless: comando, saída em JSON, worktree, armadilhas |
| | O loop de 7 passos, com a segunda rodada que ataca as decisões tomadas |
| | As armadilhas de flag entre os dois CLIs e os sinais de que a delegação virou teatro |

As duas linhas que mais mudam resultado, se você só quiser levar isso: **dar licença explícita para não achar nada** ("se não houver furo, diga isso em vez de inventar") e **fechar a saída errada** ao descrever o runtime, para o revisor não devolver o falso positivo do manual.

## Instalação

1. A skill, em nível de usuário:

   ```bash
   git clone https://github.com/mgbn01/orquestrando-claude-codex-grok-skill.git \
     ~/.claude/skills/orquestrando-claude-codex-grok
   ```

   Ou em nível de projeto, com destino `.claude/skills/orquestrando-claude-codex-grok` dentro do repositório.

2. O plugin oficial do Codex (leia o código antes e fixe a versão):

   ```text
   /plugin marketplace add https://github.com/openai/codex-plugin-cc.git#v1.0.6
   /plugin install codex@openai-codex
   /reload-plugins
   /codex:setup
   ```

3. O CLI do Grok, se for usar: [grok.com](https://grok.com), autenticado.

Não precisa chamar a skill pelo nome: o Claude Code a carrega quando o contexto bate.

## Pré-requisitos

- Plugin `codex` 1.0.6 e [`codex`](https://github.com/openai/codex) CLI 0.159.3 autenticado.
- `grok` CLI 1.0.67 autenticado.

Nenhum dos dois é obrigatório: só o Codex já dá o loop de ataque; só o Grok já dá o levantamento. As ferramentas mudam flag entre versões; se um comando falhar, confira o `--help` antes de assumir que a skill está errada.

## Procedência

A metodologia saiu de uma sessão real de endurecimento de segurança, em que o loop pegou dois achados legítimos, matou um "crítico" falso que não se reproduziu em teste e revelou um caminho de credencial desprotegido que tinha passado.

Depois ela foi testada contra um controle, um agente resolvendo o mesmo cenário **sem** a skill. O resultado, com os limites do teste, está em [`docs/teste-basico.md`](docs/teste-basico.md).

A versão 2 foi conferida em 22/10/2026 contra as ferramentas instaladas. O próprio Codex, pelo plugin, fez a revisão adversarial do texto: trouxe três divergências, as três foram reproduzidas e corrigidas antes da publicação.

## Versões

- **2.0 (22/10/2026):** integração Claude Code + Codex + Grok. Codex pelo plugin oficial, com o `codex exec` para scripts; disparo da revisão pelo próprio Claude; Grok com saída JSON e worktree; armadilhas de execução (stdin do Codex, saída embaralhada do Grok, limite de tempo); divisão de trabalho com correção pequena no Codex e braçal no Grok.
- **1.0 (01/09/2026):** orquestração pelos CLIs `grok` e `codex`.

## Licença

MIT. Veja [LICENSE](LICENSE).

Projeto independente, sem afiliação com xAI, OpenAI ou Anthropic.
