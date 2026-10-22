---
name: orquestrando-grok-e-codex
description: Use when work would benefit from a second AI engine on this machine — adversarial review of a diff or security fix, a bulk read-only codebase survey, a factual matrix that needs file:line evidence, or handing mechanical multi-file work to Grok or a small fix to Codex. Covers the official Codex plugin (/codex:review, /codex:adversarial-review, /codex:rescue) and the grok and codex CLIs. Also when the user says orquestrar, delegar, segunda opinião, revisão adversarial, grok, codex, or "effort high".
---

# Integração Claude Code + Codex + Grok

## Visão geral

O Claude Code conduz; Codex e Grok são dois motores diferentes que ele chama. O ganho não é velocidade, é **viés diferente**: um modelo treinado de outro jeito enxerga o que o seu não enxerga.

**Princípio central: eles produzem, o Claude verifica e assina.** Achado de outro motor é hipótese até ser reproduzido no código.

Como cada motor entra:

- **Codex:** pelo plugin oficial da OpenAI, [`codex-plugin-cc`](https://github.com/openai/codex-plugin-cc). Ele já resolve a chamada, o modo só leitura e os trabalhos em segundo plano. O `codex exec` fica para scripts e saída em arquivo.
- **Grok:** pelo CLI `grok` em modo headless. Não há plugin oficial da xAI para o Claude Code.

Esta skill cobre o que o plugin não traz: quem faz o quê, como escrever o pedido e como conferir o que volta.

## Divisão de papéis

| Papel | Quem | Serve para |
|---|---|---|
| **Ataque** | Codex | Revisão adversarial de um diff ou de uma decisão: segurança, corrida, perda de dados, bug de correção. |
| **Correção pequena** | Codex | Ajuste de 1 ou 2 arquivos com aceite claro (`/codex:rescue`). |
| **Levantamento** | Grok | Varredura factual em volume, só leitura e com `arquivo:linha`: inventariar rotas, tudo que uma CSP precisa permitir, todos os `<script>` inline. |
| **Braçal em vários arquivos** | Grok | Migrar um padrão em N arquivos, traduzir textos, gerar fixtures, a partir de um plano com aceite verificável. |
| **Arquitetura, decisão, teste, deploy, verificação** | Claude | Nunca delegar. É o Claude que confirma ou descarta cada achado. |

**Não delegue:** decisão de arquitetura, deploy, o teste que prova um achado, ou qualquer coisa cujo resultado você não consiga verificar.

## Escolha rápida

| Situação | Comando |
|---|---|
| Revisão normal do que mudou | `/codex:review` (branch: `--base main`) |
| Atacar uma decisão ou área de risco | `/codex:adversarial-review --base main <foco>` |
| Revisão longa sem travar a sessão | acrescente `--background`; depois `/codex:status` e `/codex:result` |
| Correção pequena delegada | `/codex:rescue --model <modelo> --effort high <tarefa>` |
| Levantamento em volume | `grok -p` com o contrato de BLOCOS (abaixo) |
| Implementação braçal em vários arquivos | `grok -p --always-approve` com plano e "git só leitura" |
| Revisão em script, com resposta em arquivo ou JSON | `codex exec -s read-only -o <arquivo>` |

## Codex pelo plugin oficial

Instalação (leia o código antes, como em qualquer plugin de terceiro, e fixe a versão revisada):

```text
/plugin marketplace add https://github.com/openai/codex-plugin-cc.git#v1.0.6
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

O `/codex:setup` diz se o CLI `codex` e o login estão prontos.

**Quem dispara o quê.** `/codex:review`, `/codex:adversarial-review`, `/codex:status` e `/codex:result` só podem ser digitados pela pessoa (`disable-model-invocation`). Quando o próprio Claude decidir pedir a revisão, ele chama o mesmo script pelo Bash:

```bash
C=$(ls ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs | tail -1)
node "$C" adversarial-review --scope working-tree "<contrato de ataque>" </dev/null > revisao.md 2>&1
```

Troque `adversarial-review` por `review` para a revisão normal; `--base main` revisa a branch. O `/codex:rescue` o Claude pode chamar direto.

- `/codex:review` e `/codex:adversarial-review` são **só leitura**. O review normal não aceita foco; o adversarial aceita texto de foco depois das flags, e é nele que entra o contrato de ataque abaixo.
- `/codex:rescue` **escreve no repositório**. Use com tarefa delimitada, arquivos nomeados e aceite; confira o diff depois.
- Modelo e esforço do review vêm do `~/.codex/config.toml`. O `/codex:review` usa `review_model` se ele existir e, se não, `model`; o adversarial usa `model`. Deixe os dois no modelo mais forte que sua conta tem (um `review_model` esquecido passa por cima do `model`), junto com `model_reasoning_effort`. No rescue, passe `--model` e `--effort` explícitos.
- O plugin traz uma revisão automática ao fim de cada resposta (`/codex:setup --enable-review-gate`). Vem desligada; ela pode prender a sessão em laço e gastar cota, então ligue só se acompanhar.

## Codex pelo CLI (scripts e saída em arquivo)

```bash
codex exec -m <modelo> -c model_reasoning_effort="high" \
     -C <caminho-do-repo> -s read-only \
     -o resposta.md "<prompt>" </dev/null
```

- `-s read-only` força investigação sem escrita; é melhor que pedir "não edite" no prompt.
- `-o/--output-last-message <arquivo>` guarda a resposta limpa. `--output-schema <arquivo>` pede JSON estruturado.
- `codex exec review` precisa de alvo: `--uncommitted`, `--base <branch>`, `--commit <sha>` ou instruções no prompt (para auditoria do repositório inteiro, diga isso no prompt). `--skip-git-repo-check` quando a pasta não é repositório.
- **Sempre `</dev/null`.** Sem ele, em segundo plano o Codex fica preso em "Reading additional input from stdin..." e nunca responde.

## Grok pelo CLI

```bash
grok -m <modelo> --reasoning-effort high --cwd <caminho-do-repo> \
     --output-format json -p "<prompt>" > grok.json
```

- `-p/--single` é o modo headless: responde e sai. **Sem `-p` ele abre a interface e trava num pipe** ("Device not configured").
- `--output-format json` traz a resposta final inteira no campo `text`. Use sempre que o prompt for grande: com `-p` puro ele abre subagentes e o texto sai embaralhado.
- Para implementar, acrescente `--always-approve` e escreva no prompt: "git só leitura, sem commit; quem commita sou eu".
- `--json-schema '<schema>'` pede saída estruturada; `--max-turns N` limita a execução.
- `-w/--worktree` combina com `-p` e isola a implementação numa worktree git (inclui as mudanças não commitadas). Não combina com `--fork-session`, e falha em perfil de sandbox que não cria worktree, como `strict`.
- Um implementador Grok por vez no mesmo working tree, e um teto diário de gasto combinado com quem paga a conta.

## Armadilhas de flag

Os dois CLIs usam as mesmas letras para coisas diferentes:

| Flag | No Grok | No Codex |
|---|---|---|
| `-p` | `--single` = o prompt headless | `--profile` = perfil de config ⚠️ |
| `-c` | `--continue` = retomar sessão ⚠️ | `--config key=value` |
| esforço | `--reasoning-effort high` | `-c model_reasoning_effort="high"` |
| pasta | `--cwd <dir>` | `-C/--cd <dir>` |

## Armadilhas de execução

- **Revisão longa morre no limite de tempo do Bash** e não escreve o `-o`. Rode desacoplada: `(nohup codex exec … </dev/null > log 2>&1 &)`, um prompt por item ou grupo pequeno, e espere o arquivo aparecer. Com o plugin, use `--background`.
- **Modelo novo dando "not supported"** costuma ser CLI desatualizado. Atualize o CLI antes de concluir que a conta não tem o modelo, e nunca no meio de uma execução.
- **Sempre fixe o modelo.** O padrão da ferramenta acha menos, e o que ele não achar você não fica sabendo que existiu.

## O loop

1. **Levantar:** o Grok varre e devolve os fatos com `arquivo:linha`.
2. **Decidir e implementar:** o Claude, com teste. Braçal verificável pode ir ao Grok.
3. **Atacar:** `/codex:adversarial-review` com o contrato de ataque.
4. **Verificar cada achado:** escrever o teste que reproduz. Não reproduziu, morreu.
5. **Endurecer:** aplicar só os achados que sobreviveram.
6. **Segunda rodada:** devolver ao Codex o diff endurecido **contando o que foi decidido e por quê**, e mandar atacar as decisões. É aqui que aparece o que a primeira rodada não pegou.
7. **Assinar:** CI verde, deploy, e a responsabilidade é de quem conduz.

Paralelize o passo 3: dispare o ataque em segundo plano (`--background` no comando, ou o script com `run_in_background`) e siga com o item seguinte.

## Contrato de prompt — Grok (levantamento)

Grok solto devolve recomendação genérica. Preso ao contrato, devolve dado:

1. **Papel, tarefa e só leitura explícito:** "Você é engenheiro de segurança web. Tarefa de LEVANTAMENTO (só leitura, não edite nada)."
2. **O stack em uma linha:** o que é o app, onde roda, se é público.
3. **"Preciso de dados concretos extraídos do código, não de recomendações genéricas."**
4. **BLOCOS numerados**, um por pergunta, cada um dizendo onde procurar. Os blocos impedem a resposta de virar ensaio.
5. **O último bloco é o entregável**, normalmente uma tabela com colunas definidas por você.
6. **Anti-alucinação:** "Cite arquivo:linha sempre. Se não encontrar, diga 'não encontrado' em vez de supor."

## Contrato de prompt — Codex (ataque)

No plugin, isto vai como texto de foco do `/codex:adversarial-review`; no CLI, como o prompt.

1. **Postura adversarial.** O prompt do plugin já faz isso; no CLI, escreva: "Você é um revisor de segurança adversarial, sem pena."
2. **O runtime e a contra-advertência.** Não basta descrever o runtime ("objeto stateful com uma instância por chave, execução serializada"). É preciso **fechar a saída errada**: "o runtime já serializa as chamadas; só aponte corrida se ela sobreviver a isso, e diga exatamente como." Sem essa linha, o revisor reinventa a corrida que o runtime já resolve.
3. **Aponte o diff, não cole.** Com acesso ao repositório, ele lê sozinho: "leia o diff E as funções inteiras que ele toca, não só as linhas mudadas."
4. **Vetores numerados que você suspeita:** contorno do bloqueio, DoS direcionado, oráculo de existência, corrida, custo de armazenamento. Numerar dirige o ataque.
5. **Licença para não achar nada:** "Se não houver furo, diga isso claramente em vez de inventar." Sem essa linha, ele fabrica achado para justificar a chamada.
6. **Teto de palavras:** "máximo 400 palavras".
7. **Na segunda rodada, entregue as decisões e mande atacá-las:** "A corrida que você apontou NÃO se reproduziu; dois testes de integração fecham a contagem certa. Ataque essa conclusão."

## A regra de ouro

**Achado de outro motor é hipótese. Reproduza antes de agir.**

Caso real (endurecimento de um fluxo de login): o Codex devolveu 3 achados, um marcado como corrida **crítica**. Ler o código dava razão a ele: o valor era lido antes do `await`. Dois testes de concorrência não reproduziram; o runtime já serializava as chamadas. O "crítico" morreu, e os outros dois eram reais: um virou ajuste de parâmetro e o outro, risco aceito e documentado. Antes disso, uma auditoria delegada tinha trazido outro "crítico" falso: credencial supostamente exposta no código, quando produção lia de um secret.

Vale nos dois sentidos: na rodada seguinte, o Codex com esforço alto achou um caminho de credencial realmente desprotegido que tinha passado.

## Red flags — pare

- Colar o achado no relatório sem ter reproduzido.
- Análise com esforço baixo. Para revisão e levantamento, esforço alto nos dois.
- Prompt de ataque sem a licença de "não achei nada".
- Prompt de ataque que descreve o runtime mas não fecha a saída errada.
- Codex sem modelo fixado, no `config.toml` ou no `-m`.
- `/codex:rescue` sem arquivos nomeados e aceite, ou sem conferir o diff depois.
- Levantamento sem blocos e sem exigir `arquivo:linha`.
- Delegar a decisão em vez do levantamento.
- `grok` sem `-p` dentro de pipe, ou `codex exec` sem `</dev/null`.
- Dois implementadores no mesmo working tree, ou delegado fazendo commit.

Conferido em 22/10/2026 com o plugin `codex` 1.0.6, `codex-cli` 0.159.3 e `grok` 1.0.67. As ferramentas mudam flag entre versões; se um comando falhar, confira o `--help` antes de assumir que a skill está errada.
