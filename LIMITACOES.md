# Limitações e bugs do BDFD

Coisas que o BDFD faz diferente do que parece, achadas testando de verdade. Cada uma tem o problema, um exemplo e o contorno.

As técnicas pra contornar a falta de loop, banco e sessão estão nas [Técnicas do Seaka](TECNICAS.md).

- [$else não pode encostar em nada](#else-não-pode-encostar-em-nada)
- [$onlyIf dentro de $try dispara o $catch](#onlyif-dentro-de-try-dispara-o-catch)

---

## $else não pode encostar em nada

O `$else` não tem colchetes. Então o BDFD só sabe onde ele termina se tiver um espaço ou uma quebra de linha **dos dois lados**.

```bdfd
$if[$var[x]==1]um $else outro $endif      ✅
$if[$var[x]==1]um$else outro$endif        ❌ colado no "um"
$if[$var[x]==1]um $elseoutro $endif       ❌ vira uma função $elseoutro, que não existe
$if[$var[x]==1]um $else$var[y;2]$endif    ❌ colado no $var
```

Normalmente é só dar um espaço. O problema é quando **o espaço atrapalha**.

### Quando precisa ficar colado

Imagine uma fileira de 8 casas, alternando ⬛ e ⬜, montada com o [laço com $eval](TECNICAS.md#laço-com-eval). As casas precisam sair grudadas: `⬛⬜⬛⬜⬛⬜⬛⬜`. Qualquer espaço no código vira espaço na fileira.

O jeito óbvio seria:

```bdfd
$var[i;1]
$var[code;$repeatMessage[8;$|if[$|modulo[$|var[i\]%|{-SEMICOL-}%2\]==1\]⬛$|else⬜$|endif$|var[i%|{-SEMICOL-}%$|calculate[$|var[i\]+1\]\]]]
$eval[$replaceText[$var[code];|;;-1]]
```

Depois que as `|` saem, cada volta vira `$if[...]⬛$else⬜$endif`: o `$else` fica colado nos dois quadrados. Com espaço (`⬛ $else ⬜`), a fileira sai com espaços no meio.

### O contorno: $elseif[1>0]

`1>0` é sempre verdade, então `$elseif[1>0]` funciona como um `$else`. A diferença é que ele **tem colchetes**, e os colchetes mostram pro BDFD onde ele começa e termina. Pode colar à vontade:

```bdfd
$var[i;1]
$var[code;$repeatMessage[8;$|if[$|modulo[$|var[i\]%|{-SEMICOL-}%2\]==1\]⬛$|elseif[1>0\]⬜$|endif$|var[i%|{-SEMICOL-}%$|calculate[$|var[i\]+1\]\]]]
$eval[$replaceText[$var[code];|;;-1]]
```

Saída: `⬛⬜⬛⬜⬛⬜⬛⬜`.

É assim que o xadrez do Sea monta o tabuleiro:

```bdfd
$|if[$|var[a\]==$|var[b\]\]$|var[a%|{-SEMICOL-}%$|var[p\]\]$|elseif[1>0\]$|var[a%|{-SEMICOL-}%$|var[b\]\]$|endif
```

> **Lembre:** o `$endif` também não tem colchetes. No exemplo ele fica colado no `$|var` seguinte e funciona, mas se der problema, a mesma ideia vale: deixe um espaço depois dele, onde o espaço não aparecer na saída.

---

## $onlyIf dentro de $try dispara o $catch

Um `$onlyIf` que falha dentro de um `$try` não só para o comando: ele **manda a mensagem dele e roda o `$catch` também**. No catch, `$error[command]` é `$onlyIf` e `$error[message]` vem vazio.

```bdfd
$try
$onlyIf[$var[aposta]>=10;A aposta mínima é 10.]
...
$catch
Deu erro: $error[message]
$setChannelVar[sala;livre]
$endtry
```

Quem apostar 5 recebe "A aposta mínima é 10." **e** "Deu erro:", e a sala é liberada, mesmo sem erro nenhum.

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

<sub>Seaka · BDFD · por <b>hirozn</b> · <a href="https://mnsa.com.br">mnsa.com.br</a></sub>
