# Limitações e bugs do BDFD

Coisas que o BDFD faz diferente do que parece. Cada uma foi testada num bot de verdade e tem o problema, um exemplo e o contorno.

As técnicas pra contornar a falta de loop, banco e sessão estão nas [Técnicas do Seaka](TECNICAS.md).

- [$else colado em outra função mata o comando](#else-colado-em-outra-função-mata-o-comando)
- [$onlyIf dentro de $try dispara o $catch](#onlyif-dentro-de-try-dispara-o-catch)
- [O código depois do $endasync roda antes do bloco](#o-código-depois-do-endasync-roda-antes-do-bloco)
- [$stop dentro de $async só encerra o bloco](#stop-dentro-de-async-só-encerra-o-bloco)
- [Resposta efêmera a um botão vem com os botões da mensagem](#resposta-efêmera-a-um-botão-vem-com-os-botões-da-mensagem)
- [Num comando de mensagem, os botões só aparecem no fim](#num-comando-de-mensagem-os-botões-só-aparecem-no-fim)

---

## Resposta efêmera a um botão vem com os botões da mensagem

Quando alguém clica num botão e o bot responde com `$ephemeral` (a mensagem que só a pessoa vê), a resposta vem com **uma cópia dos botões** da mensagem onde o clique aconteceu.

```bdfd
$if[$getTextSplitIndex[$authorID]==-1]$else $ephemeral Você já está nessa partida! $stop $endif
```

Clicou em **Entrar** duas vezes? A resposta "Você já está nessa partida!" aparece com Entrar, Iniciar e Sacar embaixo. Parece que a mensagem tem botões próprios, e quem clica neles se confunde.

### O contorno

Junte um `$removeButtons` ao `$ephemeral`. Na resposta efêmera, ele tira a cópia, e a mensagem original continua com os botões dela:

```bdfd
$if[$getTextSplitIndex[$authorID]==-1]$else $ephemeral $removeButtons Você já está nessa partida! $stop $endif
```

É por isso que o [Uno](https://github.com/seaka-bot/uno) escreve sempre `$ephemeral $removeButtons` nas recusas.

---

## Num comando de mensagem, os botões só aparecem no fim

O `$addButton` coloca o botão na **resposta** do comando, e a resposta só sai quando o comando **termina**. Se o comando espera, por exemplo com `$async` e `$replyIn`, os botões só aparecem depois da espera.

Um crash que abre a sala, espera 25 segundos e roda o jogo no mesmo comando mostra o botão **Entrar** só quando a partida já acabou.

### O contorno

Divida em dois, como no [Crash](https://github.com/seaka-bot/crash-bdfd) e no Uno:

1. O comando de mensagem só abre a sala: responde com os botões e **termina na hora**.
2. O jogo roda no `$onInteraction` do botão **Iniciar**. Comece com `$defer`, que avisa o Discord que o clique foi recebido. Sem ele, um jogo que demora mais de 3 segundos mostra "interação falhou".
3. Durante o jogo, mude a mensagem com `$editMessage[$channelID;$var[id];;título;descrição;cor]`. Com o conteúdo vazio e título e descrição preenchidos, ela vira uma embed.

---

## $else colado em outra função mata o comando

O `$else` não tem colchetes, mas aguenta texto colado dos dois lados:

```bdfd
$if[1==2]A$else B$endif                   → B
$if[1==2]A $else⬜$endif                   → ⬜
$if[1==2]primary$elsesecondary$endif      → secondary
```

O que ele **não** aguenta é outra função colada logo depois:

```bdfd
$if[1==2]A $else$var[x;B]$var[x]$endif    → nada
```

E não é um erro: o comando **nem roda**. Não sai mensagem, não sai aviso, nada. Por isso é tão difícil de achar.

### Quando precisa ficar colado

Imagine uma fileira de 8 casas montada com o [laço com $eval](TECNICAS.md#laço-com-eval), com as casas guardadas em variáveis: `$var[p]` é ⬛ e `$var[b]` é ⬜. A fileira precisa sair grudada, `⬛⬜⬛⬜⬛⬜⬛⬜`, então não dá pra ter espaço no código: qualquer espaço vira espaço na fileira.

O jeito óbvio seria:

```bdfd
$var[p;⬛]
$var[b;⬜]
$var[i;1]
$var[code;$repeatMessage[8;$|if[$|modulo[$|var[i\]%|{-SEMICOL-}%2\]==1\]$|var[p\]$|else$|var[b\]$|endif$|var[i%|{-SEMICOL-}%$|calculate[$|var[i\]+1\]\]]]
$eval[$replaceText[$var[code];|;;-1]]
```

Depois que as `|` saem, cada volta tem `$else$var[b]`: o `$else` colado no `$var`. O comando não roda. E com espaço (`$else $var[b]`), o espaço entra na fileira.

### O contorno: $elseif[1>0]

`1>0` é sempre verdade, então `$elseif[1>0]` funciona como um `$else`. A diferença é que ele **tem colchetes**, e os colchetes mostram pro BDFD onde ele termina. Pode colar o que quiser depois:

```bdfd
$var[code;$repeatMessage[8;$|if[$|modulo[$|var[i\]%|{-SEMICOL-}%2\]==1\]$|var[p\]$|elseif[1>0\]$|var[b\]$|endif$|var[i%|{-SEMICOL-}%$|calculate[$|var[i\]+1\]\]]]
```

Saída: `⬛⬜⬛⬜⬛⬜⬛⬜`.

É assim que o xadrez do Sea monta o tabuleiro:

```bdfd
$|if[$|var[a\]==$|var[b\]\]$|var[a%|{-SEMICOL-}%$|var[p\]\]$|elseif[1>0\]$|var[a%|{-SEMICOL-}%$|var[b\]\]$|endif
```

---

## $onlyIf dentro de $try dispara o $catch

Um `$onlyIf` que falha dentro de um `$try` não só para o comando: ele **manda a mensagem dele e roda o `$catch` também**. No catch, `$error[command]` é `$onlyIf` e `$error[message]` vem vazio.

```bdfd
$try
$onlyIf[1==2;mensagem do onlyIf]
$catch
catch rodou: $error[command]
$endtry
```

Saem duas mensagens: primeiro `mensagem do onlyIf`, depois `catch rodou: $onlyIf`.

No [Crash](https://github.com/seaka-bot/crash-bdfd) isso derrubava a partida dos outros: um "você já está nessa partida!" rodava o catch, que zerava a sala no meio do jogo.

### O contorno

Faça todas as conferências **antes** do `$try`. Deixe dentro dele só a parte que pode quebrar de verdade:

```bdfd
$onlyIf[$var[aposta]>=10;A aposta mínima é 10.]
$try
...
$catch
...
$endtry
```

---

## O código depois do $endasync roda antes do bloco

O bloco `$async` roda em segundo plano. Enquanto ele espera, o comando **segue** pro que vem depois do `$endasync`, e só para no `$await`.

```bdfd
$var[x;começo]
$async[a]
$replyIn[3s]
$var[x;dentro]
$endasync
$var[x;fora]
$await[a]
$var[x]
```

Resposta: `dentro`. A ordem foi: `fora` (na hora), depois `dentro` (3 segundos depois, quando o bloco rodou), depois o `$var[x]` final.

### O que fazer com isso

Se uma coisa precisa acontecer **depois** da espera, ela vai **dentro** do bloco ou **depois** do `$await`, nunca entre o `$endasync` e o `$await`.

Dá pra usar a favor: entre o `$endasync` e o `$await` é o lugar de fazer algo enquanto espera, como mandar um "⏳ aguarde..." na hora.

---

## $stop dentro de $async só encerra o bloco

```bdfd
$async[a]
$replyIn[2s]
$stop
$endasync
$await[a]
depois do stop
```

Resposta: `depois do stop`. O `$stop` saiu só do bloco, e o comando continuou depois do `$await`.

### O contorno

Pra parar o comando inteiro de dentro de um bloco, marque numa variável e pare **depois** do `$await`:

```bdfd
$async[a]
$replyIn[2s]
$if[$getChannelVar[sala]==livre]$var[parar;sim]$endif
$endasync
$await[a]
$if[$var[parar]==sim]$stop$endif
```

O loop do [Crash](https://github.com/seaka-bot/crash-bdfd) faz isso: se a sala foi zerada durante a espera, ele marca `fim` e para depois do `$await`.

---

<sub>Seaka · BDFD · por <b>hirozn</b> · <a href="https://mnsa.com.br">mnsa.com.br</a></sub>
