# Write-ups - RedVault

Write-ups dos desafios de segurança web resolvidos na plataforma [RedVault](https://github.com/lunalully/redvault), um laboratório local de segurança ofensiva. Cada entrada documenta o raciocínio por trás da exploração, não só o resultado.

---

## PRISM ARCHIVE (recon-backup-vault) - Reconhecimento - Iniciante

**Objetivo:** Encontrar uma exportação de backup exposta e recuperar o valor VAULT_ACCESS.

**Passo a passo:**
1. Li o briefing e identifiquei que "política de crawler" se referia ao arquivo `robots.txt`.
2. Acessei `http://127.0.0.1:42000/robots.txt` e encontrei duas pastas listadas como "Disallow": `/backups/` e `/admin/`.
3. Um comentário no arquivo ("maintenance export retained for replay") indicava que havia algo relevante guardado na pasta de backups.
4. Acessei `/backups/` diretamente e a pasta permitia listagem de diretório, expondo dois arquivos: `content.db.bak` e `ops.env`.
5. Abri o `ops.env` e encontrei a variável VAULT_ACCESS dentro dele.
6. Enviei o valor no formato `RV{...}` e a flag foi aceita.

**Técnica principal:** Enumeração via robots.txt + exposição de diretório (directory listing).

**O que aprendi:** Arquivos de backup e `.env` nunca deveriam ficar acessíveis publicamente. O robots.txt, mesmo pensado só para buscadores, pode revelar sem querer a existência de caminhos sensíveis para qualquer pessoa que o abra manualmente.

---

## Northstar (auth-username-enum) - Autenticação - Iniciante

**Objetivo:** Enumerar um usuário administrativo válido e recuperar a credencial fraca para acessar o console privilegiado.

**Passo a passo:**
1. Li o briefing e identifiquei que o formulário de login apresentava duas mensagens de falha diferentes (tamanho e frase), dependendo se o usuário tentado existia ou não.
2. Testei o nome de usuário "admin" e, pela diferença na resposta de erro, confirmei que essa conta existe.
3. Acessei `http://127.0.0.1:42001/wordlist.txt` e encontrei uma wordlist com possíveis senhas.
4. Testei cada senha da lista contra a conta "admin" já confirmada, até encontrar a credencial correta.
5. Fiz login e acessei o dashboard da conta admin, onde encontrei a variável de acesso.
6. Enviei o valor no formato `RV{...}` e a flag foi aceita.

**Técnica principal:** Enumeração de usuário (diferença observável na resposta de erro) + ataque de dicionário direcionado a uma conta já confirmada.

**O que aprendi:** Um formulário de login seguro deve retornar sempre a mesma mensagem de erro, não importa se o problema é o usuário ou a senha. Qualquer diferença perceptível (tamanho da resposta, texto, até tempo de carregamento) permite que um atacante descubra quais contas existem antes mesmo de tentar quebrar a senha, reduzindo o ataque de "usuário + senha" para só "senha".

---

## Segredos no bundle (recon-js-bundle) - Divulgação de Informação - Iniciante

**Objetivo:** Extrair uma API key interna de dentro do código JavaScript entregue ao navegador e usá-la para acessar um endpoint administrativo protegido por esse header.

**Passo a passo:**
1. Li o briefing e identifiquei a pista central: mesmo a equipe dizendo que "a lógica interessante fica no servidor", o navegador precisa de alguma forma se autenticar, então essa credencial precisa estar em algum lugar acessível pelo cliente.
2. Acessei `http://127.0.0.1:<porta>/static/app.js` e li o código JavaScript entregue ao navegador.
3. Encontrei, dentro de um objeto `CONFIG`, duas informações sensíveis hardcoded: a `apiKey` e o `internalEndpoint` (`/api/internal/config`).
4. Usei o `curl` para requisitar o endpoint interno, enviando a chave encontrada no header `X-API-Key`:
   ```bash
   curl -H 'X-API-Key: <chave_encontrada>' http://127.0.0.1:<porta>/api/internal/config
   ```
5. A resposta veio em JSON, contendo o valor da flag, que enviei no formato `RV{...}` e foi aceita.

**Técnica principal:** Exposição de segredo em bundle JavaScript do lado do cliente (client-side secret exposure) + abuso de autenticação via header de API key.

**Causa raiz:** Uma credencial de serviço interno foi compilada dentro de um arquivo JavaScript entregue ao navegador. Qualquer pessoa que carregasse a página conseguia ler essa chave só inspecionando o código-fonte, sem precisar de nenhuma técnica avançada.

**Mitigação (segundo o próprio lab):** Nunca entregar credenciais privilegiadas ao navegador. O caminho correto é usar um backend-for-frontend com tokens de curta duração e escopo por usuário, rotacionar qualquer chave que vaze, e adicionar varredura de segredos (secret scanning) no pipeline de build para pegar isso antes de ir para produção.

**O que aprendi:** Tudo que chega ao navegador é, por definição, público, mesmo que esteja "escondido" dentro de um arquivo `.js` minificado. Autenticação feita só por um header com uma chave fixa não é segurança de verdade se essa chave está acessível no próprio código do cliente, ela só adiciona uma etapa extra, não sigilo real. Isso reforça a ideia de que o front-end nunca deve ser tratado como fronteira de segurança.

---

## Acesso sequencial (access-idor-invoices) - Autorização - Iniciante

**Objetivo:** Autenticar como um usuário comum, explorar a numeração sequencial das faturas e acessar o registro de liquidação de outro usuário (o administrador), sem ter permissão para isso.

**Passo a passo:**
1. Li o briefing e identifiquei a falha central: o backend verifica se o usuário está **logado**, mas não verifica se a fatura pedida **pertence** a esse usuário.
2. Fiz login com a conta de teste fornecida (`alice` / `alice123`).
3. Acessei minha própria fatura e observei que o link usava um ID sequencial simples na URL (por exemplo, `/invoices/<número>`).
4. Alterei o número do ID diretamente na URL, testando outros valores sequenciais, até encontrar uma fatura que não pertencia a mim.
5. Encontrei a fatura do administrador e, dentro dela, o registro de liquidação interno com a flag.
6. Enviei o valor no formato `RV{...}` e a flag foi aceita, sem precisar de nenhuma dica.

**Técnica principal:** IDOR (Insecure Direct Object Reference) — referência direta e insegura a objeto, explorada pela enumeração de IDs sequenciais.

**Causa raiz:** A aplicação confunde **autenticação** (saber quem é o usuário) com **autorização** (verificar se esse usuário tem permissão sobre o recurso específico que está pedindo). Como os IDs são sequenciais e previsíveis, bastou incrementar ou trocar o número na URL para acessar dados de outra conta.

**O que aprendi:** Estar logado não deveria ser suficiente para acessar qualquer recurso do sistema, o backend precisa sempre confirmar que o dono da sessão é também o dono do dado pedido. IDs sequenciais e previsíveis (1, 2, 3...) tornam esse tipo de falha trivial de explorar, por isso identificadores mais aleatórios (como UUIDs) dificultam, ainda que sozinhos não resolvam o problema de autorização.

## A porta booleana (auth-sql-boolean) - SQL Injection - Iniciante

**Objetivo:** Fazer bypass da query de login da Parsec através de SQL Injection, alcançando uma sessão administrativa sem conhecer a senha real.

**Passo a passo:**
1. Li o briefing e identifiquei o ponto fraco: a query de login é montada por **concatenação de strings**, sem parametrização, o que significa que o que eu digito nos campos vai direto para dentro do comando SQL.
2. Entendi que a query provavelmente tinha a forma `WHERE username = '<input>' AND password = '<input>'`, controlando os dois lados da condição.
3. Usei o nome de usuário `' OR 1=1 --` no campo de login, com qualquer valor no campo de senha.
4. Essa entrada fecha a aspas da string do `username`, acrescenta a condição `OR 1=1` (sempre verdadeira) e comenta (`--`) o restante da query, incluindo a verificação de senha.
5. Com a cláusula `WHERE` sempre verdadeira, o login retornou a primeira linha da tabela de usuários, que era a conta `admin`.
6. Acessei a página autenticada e enviei a flag exibida, no formato `RV{...}`, que foi aceita.

**Técnica principal:** SQL Injection (bypass de autenticação via manipulação booleana da cláusula WHERE).

**Causa raiz:** A aplicação constrói a query SQL por concatenação direta das entradas do usuário, em vez de usar *queries parametrizadas* (prepared statements). Isso permite que qualquer caractere especial de SQL, como aspas simples, altere a lógica da consulta original.

**O que aprendi:** Nunca concatenar entrada do usuário diretamente numa query SQL, por mais "interna" ou "legada" que a tela de login pareça ser. Mesmo sem saber nenhuma senha, só manipulando a sintaxe da própria query foi possível transformar uma condição de autenticação em algo sempre verdadeiro. A defesa correta é sempre usar consultas parametrizadas, que tratam a entrada do usuário como dado puro, nunca como parte do código SQL.
