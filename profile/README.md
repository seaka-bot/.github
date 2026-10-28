<div align="center">

# Sea か

**Um bot de Discord feito em BDFD, entre 2023 e 2024.**
Jogos multiplayer, economia, perfil e um sistema visual próprio, tudo sem servidor, sem banco de dados

[Portfólio](https://mnsa.com.br) · [Estudos de caso jogáveis](https://mnsa.com.br/#trabalhos)

</div>

---

> [!WARNING]
> **Isto é um acervo, não um projeto ativo.**
> O código aqui está parado. Ele foi escrito pra uma versão do BDFD que já mudou, então algumas funções podem não existir mais ou se comportar diferente. Não tem suporte, não aceito pull requests de correção e não garanto que nada rode do jeito que está.
> Guardei porque as **ideias** ainda servem. Se você está aprendendo BDFD, ou quer ver como resolver problemas difíceis com uma ferramenta limitada, pode ser útil.

## O que é o BDFD

**BDFD (Bot Designer for Discord)** é um aplicativo pra criar bots de Discord sem montar servidor. Você escreve os comandos no app, pelo celular ou pelo navegador, e o próprio BDFD hospeda e roda o bot.

Os comandos são escritos em **BDScript**, uma linguagem feita só de funções que começam com `$`:

```
$nomention
$title[Olá!]
$description[Você tem $getUserVar[moedas] moedas.]
$addButton[no;jogar;Jogar;primary]
```

Cada função faz uma coisa: lê uma variável, manda uma mensagem, cria um botão. O bot é a soma de dezenas de comandos assim.

**Onde roda:** nos servidores do BDFD. Não existe arquivo pra baixar e rodar no seu computador. Pra usar qualquer código daqui, você precisa colar ele dentro de um bot criado no app do BDFD.

**As limitações que moldaram este código** (na época em que foi escrito):
- não tinha laço de repetição (`while`, `for`);
- não tinha banco de dados, só variáveis de texto por usuário, canal ou servidor;
- não tinha sessão: cada clique de botão é um comando novo, que não lembra do anterior.

Boa parte do código aqui existe pra contornar essas três coisas.

## Créditos e licença

- **Código:** [MIT — use, copie e adapte, só mantenha o crédito].
- **Cartas do Uno, logo e banner:** desenhados por mim, hirozn. [MIT — use, copie e adapte, só mantenha o crédito]
- **Ícones das figurinhas:** desenhos do [Flaticon](https://www.flaticon.com), de autores diversos, adaptados pro sistema visual do bot. Eles **não** estão sob a licença do código; seguem a licença do Flaticon.

---

<div align="center">

Feito por **hirozn** · Matheus · [mnsa.com.br](https://mnsa.com.br)

</div>
