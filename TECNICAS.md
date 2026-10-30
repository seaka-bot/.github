# Técnicas do Seaka

O BDFD não tinha loop, nem banco de dados, nem sessão. Todo projeto do Seaka usa as mesmas gambiarras pra contornar isso, então elas estão explicadas aqui, uma vez só.

Os exemplos são do código real do [Uno](https://github.com/seaka-bot/uno) e do [Crash](https://github.com/seaka-bot/crash-bdfd).

> Pegadinhas do próprio BDFD, como o `$else` que não pode encostar em nada, estão em [Limitações e bugs](LIMITACOES.md).

- [Laço com $eval](#laço-com-eval): repetir comandos sem ter loop
- [Espera com $async](#espera-com-async): um laço que espera entre uma volta e outra
- [Emojis dinâmicos](#emojis-dinâmicos): criar uma variável pra cada emoji, sozinho
- [String como banco de dados](#string-como-banco-de-dados): guardar listas numa variável de texto
- [Estado no ID do botão](#estado-no-id-do-botão): saber de quem é o clique sem ter sessão
- [Trava por tempo](#trava-por-tempo): impedir duas partidas no mesmo canal

---

## Laço com $eval

Um loop, em outras linguagens, serve pra repetir ações automaticamente. O BDFD não tem uma função de loop, mas dá pra conseguir o mesmo resultado com uma estratégia engenhosa.

Por exemplo, considere este código:

```bdfd
$sendMessage[oi1]
$sendMessage[oi2]
$sendMessage[oi3]
$sendMessage[oi4]
$sendMessage[oi5]
...
$sendMessage[oi10]
```

Em vez de escrever cada linha à mão, dá pra fazer assim:

```bdfd
$var[i;1]
$var[loop;$repeatMessage[10;$|sendMessage[oi $|var[i\]] $|var[i\;$|sum[$|var[i\];1\]]]]
$eval[$replaceText[$var[loop];|;;-1]]
```

### Explicação

1. **`$repeatMessage`**
   Repete um texto várias vezes. Só que ele não repete comandos, como `$sendMessage`.

2. **Transformar em texto**
   Escrevendo `$|sendMessage` em vez de `$sendMessage`, o comando vira texto, e aí o `$repeatMessage` consegue repetir.

3. **A variável `i`**
   É o contador. Começa em 1, e no fim de cada volta soma 1 até chegar em 10.

4. **Caracteres especiais**
   Como isso é uma gambiarra, dentro do loop todo `]` e `;` precisa de uma barra invertida antes, senão o loop roda antes da hora:
   ```
   \]
   \;
   ```
   No código do Uno você também vai ver `%|{-SEMICOL-}%`. Depois que as `|` saem, ele vira `%{-SEMICOL-}%`, que o BDFD lê como `;`. É outro jeito de escrever um `;` que só vale depois do `$eval`.

5. **`$eval` e `$replaceText`**
   O `$replaceText` apaga todas as `|`, o texto volta a ser comando, e o `$eval` executa.

> Esse truque não serve só pra `$sendMessage`: dá pra repetir qualquer comando.

> **Dica:** se o loop não fizer o que você espera, mostre `$var[loop]` antes do `$eval`. Você vê exatamente o código que foi montado.

---

## Espera com $async

O [laço com $eval](#laço-com-eval) roda todas as voltas de uma vez. No Crash isso não serve: o gráfico precisa subir **um nível por segundo**, e as pessoas precisam conseguir sacar no meio.

O BDFD não tem `sleep`. Mas tem o `$async`:

```bdfd
$async[espera]
$replyIn[1s]
$sendMessage[passou 1 segundo]
$endasync
$await[espera]
```

1. **`$async[nome]` … `$endasync`**
   Marca um bloco que roda em segundo plano.

2. **`$replyIn[1s]`**
   Dentro do bloco, atrasa o bloco em 1 segundo.

3. **`$await[nome]`**
   Segura o comando até aquele bloco acabar. O comando fica parado 1 segundo, e só então continua.

Juntando com o laço, cada volta vira um bloco com o número da volta como nome. Este é o esqueleto do Crash:

```bdfd
$var[i;1]$var[code;$repeatMessage[10;
$|async[$|var[i\]\]
$|replyIn[1s\]
$|var[r%|{-SEMICOL-}%$|random[1%|{-SEMICOL-}%100\]\]
$|endasync
$|await[$|var[i\]\]
$|editMessage[$|channelID%|{-SEMICOL-}%$|var[id\]%|{-SEMICOL-}%nível $|var[i\]\]
$|var[i%|{-SEMICOL-}%$|calculate[$|var[i\]+1\]\]
]]
$var[code;$repeatMessage[2;$var[code]]]
$eval[$replaceText[$var[code];|;;-1]]
```

### Explicação

1. **`$|async[$|var[i\]\]`**
   Cada volta abre um bloco com nome diferente: `1`, `2`, `3`… Assim o `$await` de uma volta não confunde com o da outra.

2. **`$|replyIn[1s\]` e `$|await`**
   A volta espera 1 segundo antes de sortear o nível.

3. **`$|editMessage`**
   Em vez de mandar uma mensagem por volta, edita sempre a mesma (`$var[id]`, guardada com `$sendMessage[...;yes]` antes do laço). O gráfico "anda" na mesma mensagem.

4. **`$repeatMessage[2;$var[code]]`**
   Dobra o código pronto: 10 voltas × 2 = 20 níveis. Pedir 20 de uma vez no primeiro `$repeatMessage` deixaria o texto montado bem maior.

### Enquanto um comando espera, os outros rodam

Essa é a parte boa. Durante o segundo de espera, outra pessoa pode mandar `s*sair`. Esse comando roda na hora e muda a variável dela:

```bdfd
$setVar[crash;false-0-0;$authorID]
```

Na volta seguinte, o laço do Crash confere essa variável antes de mexer no jogador. Quem já saiu é pulado:

```bdfd
$|if[$|checkContains[$|getVar[crash%|{-SEMICOL-}%$|var[p\]\]%|{-SEMICOL-}%true\]==true\]
```

Ou seja: **dois comandos conversam por uma variável**. Um fica em laço, lendo. O outro roda no meio da espera e escreve.

> **Cuidado:** leia a variável **dentro** do bloco, depois do `$replyIn`. Se ler antes da espera, o laço vê o valor velho, de antes de a pessoa sacar.

> **Lembre:** o comando que espera continua ocupando o bot. Com 20 níveis de 1 segundo, ele roda por uns 20 segundos. Prefira poucas voltas e espera curta.

---

## Emojis dinâmicos

No Uno, cada carta é um emoji que você sobe num servidor do Discord. Quando você sobe, dá um **nome** pra ele, por exemplo `r1` pro 1 vermelho.

Só que, pra usar um emoji customizado no código, o Discord exige o nome **e o ID**:

```bdfd
$var[r1;<:r1:1128484025653280800>]
$var[r2;<:r2:1128484025653280801>]
$var[r3;<:r3:1128484025653280802>]
...
$var[0b;<:0b:1128484025653280850>]
```

São 50 linhas, uma por carta. E se você trocar o desenho de uma carta, o ID muda e você precisa editar o código de novo.

Em vez disso, o bot pode **ler os emojis do servidor sozinho** e criar uma variável pra cada um, **usando o nome do emoji como nome da variável**. É este o código do Uno:

```bdfd
$var[i;1]$var[code;$repeatMessage[10;$|textSplit[$|serverEmojis[$|getVar[UnoEmojis%|{-SEMICOL-}%$|botID\]%|{-SEMICOL-}%-\]%|{-SEMICOL-}%-\]$|textSplit[$|splitText[$|var[i\]\]%|{-SEMICOL-}%:\]$|var[nome%|{-SEMICOL-}%$|splitText[2\]\]$|textSplit[$|serverEmojis[$|getVar[UnoEmojis%|{-SEMICOL-}%$|botID\]%|{-SEMICOL-}%-\]%|{-SEMICOL-}%-\]$|var[$|var[nome\]%|{-SEMICOL-}%$|splitText[$|var[i\]\]\]$|var[i%|{-SEMICOL-}%$|calculate[$|var[i\]+1\]\]]]$var[code;$repeatMessage[5;$var[code]]]$eval[$replaceText[$var[code];|;;-1]]
```

Parece assustador, mas é o mesmo [laço com $eval](#laço-com-eval). Tirando as `|` e trocando `%{-SEMICOL-}%` por `;`, **cada volta do laço** executa isto:

```bdfd
$textSplit[$serverEmojis[$getVar[UnoEmojis;$botID];-];-]
$textSplit[$splitText[$var[i]];:]
$var[nome;$splitText[2]]
$textSplit[$serverEmojis[$getVar[UnoEmojis;$botID];-];-]
$var[$var[nome];$splitText[$var[i]]]
$var[i;$calculate[$var[i]+1]]
```

### Explicação

1. **`$getVar[UnoEmojis;$botID]`**
   É o ID do servidor onde estão os emojis das cartas. Ele fica guardado numa variável do bot.

2. **`$serverEmojis[...;-]` + `$textSplit[...;-]`**
   `$serverEmojis` devolve todos os emojis desse servidor, separados por `-`:
   ```
   <:r1:1128…>-<:r2:1128…>-<:r3:1128…>
   ```
   O `$textSplit` quebra essa lista, um emoji por pedaço.

3. **`$textSplit[$splitText[$var[i]];:]`**
   Pega o emoji número `i` e quebra ele nos `:`:
   ```
   <:r1:1128…>   →   <   r1   1128…>
   ```

4. **`$var[nome;$splitText[2]]`**
   O pedaço 2 é o **nome do emoji**: `r1`.

5. **`$textSplit[$serverEmojis[...]]` de novo**
   O passo 3 trocou o que estava quebrado, então quebra a lista de emojis outra vez pra conseguir pegar o emoji inteiro.

6. **`$var[$var[nome];$splitText[$var[i]]]`**
   Essa é a parte dinâmica. O nome da variável não está escrito no código: ele vem do `$var[nome]`, que é o nome do emoji. Então essa linha vira:
   ```bdfd
   $var[r1;<:r1:1128…>]
   ```

7. **`$var[i;...+1]`**
   Passa pro próximo emoji.

8. **`$repeatMessage[10;...]` e depois `$repeatMessage[5;$var[code]]`**
   Monta 10 voltas e repete esse bloco 5 vezes: 10 × 5 = **50 voltas**, ou seja, até 50 emojis.

### O resultado

**Cada emoji do servidor vira uma variável com o nome dele.** Se o servidor tem 41 emojis, o bot cria 41 variáveis. Se você subir um emoji chamado `r1`, passa a existir a variável `$var[r1]`, e escrever isso no código mostra o emoji.

No Uno, a carta da mesa fica salva como texto, por exemplo `r5`, então pra mostrar o emoji dela basta:

```bdfd
$var[$getChannelVar[UnoCarta]]
```

O bot lê `r5`, vira `$var[r5]` e aparece o emoji da carta.

### Pra funcionar

- Crie um servidor **só** pros emojis daquele jogo. O laço lê no máximo 50.
- Dê aos emojis **exatamente** os nomes que o código usa (cada projeto lista os seus).
- Guarde o ID desse servidor na variável (no Uno, `UnoEmojis` do bot).

> Trocou o desenho de uma carta? É só subir de novo **com o mesmo nome**. O jogo pega o novo sozinho, sem mexer no código.

---

## String como banco de dados

O BDFD não tem banco de dados, só variáveis de texto. Então como guardar quem está numa partida?

Tudo numa variável só, separado por `-`. A variável `UnoParty` de um canal fica assim:

```
111-222-333
```

São 3 jogadores (pelo ID do Discord), e o primeiro é o host.

### Exemplo 1: entrar na party

Este é o código do botão **Entrar na Party**:

```bdfd
$textSplit[$getChannelVar[UnoParty];-]
$if[$getTextSplitIndex[$authorID]!=-1]$ephemeral $removeButtons Você já está na party. $stop $endif
$setChannelVar[UnoParty;$joinSplitText[-]-$authorID]
```

1. **`$textSplit[$getChannelVar[UnoParty];-]`**
   Quebra a lista nos `-`. Agora cada jogador é um pedaço: 1, 2, 3.

2. **`$getTextSplitIndex[$authorID]`**
   Procura quem clicou na lista. Devolve a posição dele, ou `-1` se ele não estiver lá.

3. **`!=-1` → "Você já está na party."**
   Se achou, a pessoa já está na party. Responde só pra ela (`$ephemeral`) e para (`$stop`).

4. **`$joinSplitText[-]-$authorID`**
   Junta a lista de volta com `-` e cola o ID novo no fim:
   ```
   111-222-333   →   111-222-333-444
   ```

5. **`$setChannelVar[UnoParty;...]`**
   Salva de volta na variável.

### Exemplo 2: remover um jogador

Este é o botão **Remover** do painel de host:

```bdfd
$textSplit[$getChannelVar[UnoParty];-]
$if[$var[a]==1]$ephemeral $removeButtons Você não pode remover o host da partida.$stop $endif
$removeSplitTextElement[$var[a]]
$setChannelVar[UnoParty;$joinSplitText[-]]
```

1. **`$var[a]`** é a posição do jogador selecionado no painel.
2. **`$var[a]==1`**: a posição 1 é o host, e ele não pode ser removido.
3. **`$removeSplitTextElement[$var[a]]`** tira o jogador dessa posição.
4. **`$joinSplitText[-]`** junta de novo e salva:
   ```
   111-222-333   →   111-333
   ```

### E pra guardar mais de uma coisa?

Usa dois separadores. No Uno, a mão de todo mundo fica na variável `UnoCartas`, assim:

```
r5-g2-0c/b3-yx/r9
```

O `/` separa os jogadores e o `-` separa as cartas. O jogador 1 tem `r5 g2 0c`, o 2 tem `b3 yx` e o 3 tem `r9`.

Este é o código que tira uma carta da mão de quem jogou:

```bdfd
$textSplit[$getChannelVar[UnoCartas];/]
$textSplit[$splitText[$var[pos]];-]
$removeSplitTextElement[$var[id]]
$var[edit;$joinSplitText[-]]
$textSplit[$getChannelVar[UnoCartas];/]
$editSplitText[$var[pos];$var[edit]]
$setChannelVar[UnoCartas;$joinSplitText[/]]
```

1. **Quebra no `/`** e pega a mão do jogador `pos`: `b3-yx`.
2. **Quebra essa mão no `-`**: `b3`, `yx`.
3. **`$removeSplitTextElement[$var[id]]`** tira a carta jogada (posição `id` na mão).
4. **Junta a mão com `-`** e guarda em `$var[edit]`: `yx`.
5. **Quebra tudo de novo no `/`**, porque o passo 2 trocou o que estava quebrado.
6. **`$editSplitText[$var[pos];$var[edit]]`** troca a mão antiga do jogador pela nova.
7. **Junta com `/` e salva**:
   ```
   r5-g2-0c/b3-yx/r9   →   r5-g2-0c/yx/r9
   ```

> **Cuidado:** o separador nunca pode aparecer dentro do dado. Um `-` no meio de um nome quebra tudo. Por isso guardamos IDs e códigos de carta, não nomes.

> **Não esquece:** o `$textSplit` trabalha numa cópia. Se mudou algo, salve com `$joinSplitText` e `$setChannelVar`.

---

## Estado no ID do botão

No Discord, cada clique num botão chega pro bot como um **comando novo**. O bot não lembra de nada: não sabe de qual jogo é o botão, nem de quem é a vez.

A solução é guardar essas informações **dentro do ID do botão**. Este é o botão **Jogar** do Uno:

```bdfd
$addButton[no;unoStart-jogar-$splitText[$var[pos]]-$var[pos];Jogar;secondary]
```

O ID dele fica assim:

```
unoStart-jogar-222-2
```

| Pedaço | Valor | Significa |
|---|---|---|
| 1 | `unoStart` | qual jogo |
| 2 | `jogar` | o que fazer |
| 3 | `222` | ID de quem é a vez |
| 4 | `2` | posição dele na party |

Quando alguém clica, o comando de interação (`$onInteraction`) lê esse ID de volta:

```bdfd
$textSplit[$customID;-]
$onlyIf[$splitText[1]==unoStart; ]
$var[função;$splitText[2]]
$var[pos;$splitText[4]]
$textSplit[$getChannelVar[UnoParty];-]
$if[$and[$splitText[$var[pos]]!=$authorID;$var[função]!=host]==true]$ephemeral $removeButtons Aguarde $nickname[$splitText[$var[pos]]]. $stop $endif
```

### Explicação

1. **`$textSplit[$customID;-]`**
   `$customID` é o ID do botão que foi clicado. Quebra ele nos `-`.

2. **`$onlyIf[$splitText[1]==unoStart; ]`**
   Se o botão não é do Uno, ignora. Assim vários jogos convivem no mesmo bot.

3. **`$var[função;...]` e `$var[pos;...]`**
   Guarda o que fazer (`jogar`) e a posição de quem é a vez (`2`).

4. **`$textSplit[$getChannelVar[UnoParty];-]`**
   Abre a lista de jogadores (veja [string como banco de dados](#string-como-banco-de-dados)).

5. **`$splitText[$var[pos]]!=$authorID`**
   Confere se quem clicou é mesmo o jogador da vez. Se não for, responde só pra ele: *"Aguarde Fulano."*

> **Lembre:** o ID do botão tem limite de tamanho no Discord. Guarde só o essencial: o jogo, a ação, um ID e uma posição.

---

## Trava por tempo

Sem cuidado, duas pessoas podem abrir duas partidas no mesmo canal, e uma bagunça a outra. Também dá pra clicar num botão de uma partida antiga, que já acabou.

O Uno resolve isso guardando numa variável de canal **até quando a partida vale** e **qual mensagem é a dela**:

```
1727000000-1153000000000000000
```

O primeiro pedaço é um horário (em segundos), e o segundo é o ID da mensagem do jogo.

> A variável se chama `UnoEmojis` por motivos históricos. Na prática, no canal, ela é a trava da partida.

### Abrindo a partida

```bdfd
$textSplit[$getChannelVar[UnoEmojis];-]
$if[$splitText[1]>=$getTimestamp[s]]
$description[Aguarde o [jogo\](https://discord.com/channels/$guildID/$channelID/$splitText[2]) terminar.
Mas você poderá iniciar um novo jogo <t:$splitText[1]:R>]
$stop
$endif
...
$setChannelVar[UnoEmojis;$sum[$getTimestamp[s];300]-$var[UnoID]]
```

1. **`$getTimestamp[s]`** é o horário de agora, em segundos.
2. **`$splitText[1]>=$getTimestamp[s]`**: se a trava ainda não venceu, já tem jogo rolando.
3. **A mensagem de aviso** traz um link pro jogo que está rolando (`$splitText[2]`) e mostra quanto falta pra liberar (`<t:...:R>` vira "em 3 minutos" no Discord).
4. **`$sum[$getTimestamp[s];300]-$var[UnoID]`**: ao abrir um jogo novo, trava por 300 segundos (5 minutos) e guarda o ID da mensagem do jogo.

### Recusando botões velhos

Todo botão do Uno começa conferindo isto:

```bdfd
$textSplit[$getChannelVar[UnoEmojis];-]
$if[$splitText[2]!=$messageID]$ephemeral $removeButtons Sinto muito, mas não reconheci o ID dessa party. $stop $endif
```

`$messageID` é a mensagem onde o botão foi clicado. Se não for a mensagem do jogo atual, o botão é de uma partida velha, e o bot recusa.

### Fechando a sala

```bdfd
$setChannelVar[UnoEmojis;1-1]
```

O horário `1` já passou faz tempo, então o canal fica livre na hora.

---

<sub>Seaka · BDFD · por <b>hirozn</b> · <a href="https://mnsa.com.br">mnsa.com.br</a></sub>
