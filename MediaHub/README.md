# MediaHub (TryHackMe)

Write-up de um CTF de aplicação web: um portal interno fictício usado por jornalistas, onde o objetivo era escalar de visitante anônimo até execução remota de comando no servidor, usando o Burp Suite para interceptar e manipular requisições HTTP.

**Técnicas:** Information Disclosure, Broken Authentication, Mass Assignment, SSRF → Command Injection, Reverse Shell

## Cadeia de ataque

```
Gobuster encontra backup exposto
  → Backup revela e-mail + padrão de senha
    → Login revela etapa de 2FA
      → Mass Assignment bypassa o 2FA
        → Acesso ao painel admin
          → SSRF com filtro fraco
            → Command Injection → Reverse Shell
```

---

## Setup do ambiente

**Camada OSI:** 7 (Aplicação) — a interceptação ocorre entre navegador e servidor, no nível HTTP.

Configuração do navegador para rotear pelo Burp Suite (proxy em `127.0.0.1:8080`), com o certificado CA do Burp importado no navegador para interceptar HTTPS sem erro de certificado.

---

## 1. Reconhecimento

```bash
nmap -sV -sC <IP>
```

Portas abertas: SSH, DNS e um servidor web Apache.

```bash
gobuster dir -u http://<IP>/ -w <wordlist> --exclude-length <tamanho>
```

- `dir` — modo de enumeração de diretórios
- `--exclude-length` — ignora respostas com esse tamanho de bytes; necessário porque o servidor respondia "200 OK" até para URLs inexistentes, o que confundia a ferramenta por padrão.

O resultado trouxe alguns diretórios com listagem aberta (expondo bibliotecas JS/CSS e um avatar padrão) e um painel de administração de banco de dados exposto.

---

## 2. Information Disclosure — arquivo de backup

**Camada OSI:** 7 (Aplicação)

Testando a extensão `.bak` sobre um arquivo PHP já conhecido da aplicação:

```
http://<IP>/login.php.bak
```

O servidor serviu o arquivo como download em texto puro, em vez de processá-lo como PHP — um erro clássico de deploy, quando um arquivo de backup é deixado dentro da pasta pública. O conteúdo revelou um comentário de desenvolvedor com e-mail de administrador e o padrão usado nas senhas do sistema.

> **Lição:** sempre testar extensões alternativas (`.bak`, `.old`, `.swp`, `~`) em cima de arquivos conhecidos da aplicação. Existem wordlists específicas para isso.

---

## 3. Broken Authentication

Com o padrão de senha revelado, o login funcionou de primeira — mas a aplicação exigia uma segunda etapa de verificação (OTP de 6 dígitos).

Duas tentativas de bypass que **não** funcionaram, e que vale documentar:

- SQL Injection no campo de e-mail → resposta idêntica de "credenciais inválidas", sugerindo uso de *prepared statements* no backend.
- Manipulação do tipo do campo de senha (enviando um array em vez de texto) → também sem sucesso, sugerindo comparação de senha feita com função seria (não vulnerável à comparação solta do PHP).

> Registrar o que **não** funcionou também ajuda a entender quais defesas a aplicação implementou corretamente.

---

## 4. Mass Assignment — bypass do 2FA

**Camada OSI:** 7 (Aplicação)
**Ferramenta:** Burp Suite

A tela de verificação do código manda uma requisição simples com o campo do OTP. Em vez de tentar adivinhar o código (o que seria lento e bloqueado por *throttling* do proxy), a abordagem foi adicionar um **parâmetro extra**, não previsto pelo formulário original, direto no corpo da requisição via Burp Repeater:

```
otp=123456
is_verified=true
```

O backend confiava nesse campo `is_verified` vindo do cliente, sem de fato validar o código no servidor. Isso é uma falha de **Mass Assignment** (também chamada de *Parameter Pollution* nesse contexto): o servidor aceita e processa campos que o cliente não deveria poder controlar.

Como a sessão era compartilhada entre o Burp e o navegador, bastou recarregar a página do painel para acessar como administrador.

> **Camada OSI:** essa manipulação ocorre inteiramente na Camada 7 — o servidor confia cegamente em dados vindos do cliente, violando o princípio de nunca confiar em input do usuário.

---

## 5. SSRF → Command Injection

No painel admin havia uma funcionalidade de importar conteúdo a partir de uma URL externa (um "feed"). O servidor buscava e retornava o conteúdo bruto dessa URL.

### Testes de bypass do filtro de SSRF

| Payload | Resultado |
|---|---|
| Endereço de loopback padrão (`127.0.0.1`) | Bloqueado — filtro reconhecia o texto literal |
| Representação alternativa do mesmo endereço | Passou — o filtro só bloqueava a escrita literal, não validava o IP de fato |
| Esquema de leitura de arquivo local, sozinho | Erro — esquema não aceito puro |
| Representação alternativa + separador de comando + leitura de arquivo local | Sucesso total |

O payload final indicou que o backend provavelmente monta um comando de shell (tipo `curl <url>`) sem sanitizar o input. O separador de comando (`;`) encerra o primeiro comando e injeta um segundo, que o próprio processo interpreta como leitura de arquivo local. Isso caracteriza **Command Injection** disfarçado de SSRF — não é "só" um SSRF, é execução de comando no sistema operacional do servidor.

> **Camada OSI:** o SSRF em si ocorre na Camada 7, mas a causa raiz é Command Injection no sistema operacional do host — fora do modelo OSI, no nível de execução de processo.

---

## 6. Escalando para Reverse Shell

Confirmada a execução de comando via leitura de arquivo, o próximo passo foi trocar a leitura por uma reverse shell de verdade.

**Servidor HTTP local, servindo um script de reverse shell:**

```bash
python3 -m http.server <porta>
```

**Listener esperando a conexão:**

```bash
nc -lvnp <porta>
```

- `-l` — modo listen (escuta)
- `-v` — verbose
- `-n` — não resolve DNS
- `-p` — porta onde escuta

Usando o mesmo separador de comando do SSRF, o payload final forçou o servidor a baixar o script (via `wget`) e executá-lo em seguida (via `sh`), nessa ordem — a primeira tentativa, usando substituição de comando diretamente na URL, não funcionou, porque o backend não repassava a string inteira por um shell completo.

**Resultado no listener:** conexão recebida, shell interativa obtida como o usuário padrão do Apache (`www-data`).

> **Camada OSI:** a conexão reversa em si ocorre via TCP (Camada 4), mas a causa raiz continua sendo Command Injection na Camada 7.

---

## Conceitos-chave

- **Mass Assignment** — quando o backend aceita e processa campos extras enviados pelo cliente, sem que o formulário original os previsse, permitindo contornar validações (como aqui, o 2FA).
- **SSRF (Server-Side Request Forgery)** — forçar o servidor a fazer uma requisição por você, alcançando recursos que não seriam acessíveis de fora.
- **Command Injection** — quando a entrada do usuário é usada para montar um comando de shell sem sanitização, permitindo injetar comandos adicionais via separadores (`;`, `&&`, `|`).

## Lições aprendidas

- Arquivos de backup (`.bak`, `.old`, `~`) são um dos achados mais comuns e valiosos em enumeração de diretórios — nunca pular esse teste.
- Documentar tentativas que **não** funcionaram também tem valor: mostra quais defesas a aplicação implementou corretamente.
- Filtros de SSRF baseados em comparação de texto literal (em vez de validação real do IP resolvido) são fáceis de contornar com representações alternativas do mesmo endereço.
- Depois de confirmar Command Injection, mesmo que só por leitura de arquivo, o passo natural é escalar para uma shell interativa completa.
