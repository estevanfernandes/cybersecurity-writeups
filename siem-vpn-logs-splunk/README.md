# Investigação de Logs de VPN com Splunk (TryHackMe)

Write-up de blue team: investigação de um conjunto de logs de VPN usando Splunk, simulando o tipo de pergunta que um analista de SOC recebe no dia a dia — "quantos acessos esse usuário teve?", "de onde veio esse IP?", "quantos eventos vieram de fora deste país?".

**Plataforma:** TryHackMe — trilha SOC Level 1
**Técnicas:** Ingestão de dados em SIEM, SPL (Search Processing Language), investigação de logs

## Contexto

Diferente de um CTF ofensivo, aqui o objetivo não é comprometer um sistema, e sim **extrair informação de um conjunto de logs já coletado**, que é justamente o trabalho de quem opera um SIEM: dados brutos entram, e cabe ao analista fazer as perguntas certas pra transformar isso em informação útil (identificar um usuário suspeito, confirmar um acesso, medir o volume de eventos de uma origem).

**Camada OSI:** o SIEM em si não pertence a uma camada só — ele correlaciona dados de rede (IP de origem, camada 3) com dados de aplicação (usuário, camada 7), e é isso que dá visibilidade do ambiente como um todo.

---

## 1. Ingestão dos dados

Os dados vieram em um arquivo de log de VPN em formato **JSON**, delimitado por quebras de linha (cada linha é um evento independente).

Passos no Splunk:

1. Em **Adicionar Dados → Carregar**, selecionar o arquivo de log.
2. Manter o tipo de origem detectado automaticamente como JSON.
3. Em **Configurações de Entrada**, criar um novo índice, nomeado `VPN_Logs` — no Splunk, um índice funciona como uma "gaveta" onde os dados ficam guardados depois de importados, e é nela que as buscas futuras vão apontar.
4. Concluir o upload e, na tela de pesquisa, definir o intervalo de tempo como "Todo o período", já que os eventos do arquivo não necessariamente caem dentro da janela padrão (últimas 24h).

**Verificação da ingestão:**

```spl
index=VPN_Logs
| stats count
```

- `index=VPN_Logs` — restringe a busca a esse índice específico
- `|` (pipe) — encadeia o resultado de um comando para o próximo, como no terminal Linux
- `stats count` — conta quantos eventos existem no resultado

**Resultado:** 2.862 eventos no total, confirmando que a importação ocorreu corretamente.

> **Observação técnica:** como os dados vieram em JSON, os campos (`UserName`, `Source_ip` etc.) não aparecem automaticamente nos resultados de busca. É necessário adicionar o comando `spath` após a busca base, que instrui o Splunk a interpretar e extrair os campos JSON de cada evento.

---

## 2. Investigação por usuário

**Pergunta:** quantos eventos de log pertencem a um usuário específico?

```spl
index=VPN_Logs
| spath
| search UserName="Maleena"
| stats count
```

- `spath` — extrai os campos do JSON de cada evento
- `search UserName="Maleena"` — filtra apenas os eventos em que esse campo bate com esse valor
- `stats count` — conta os eventos restantes

**Resultado:** 60 eventos associados a essa usuária. Esse tipo de busca é o equivalente a um analista responder "quantas vezes essa pessoa acessou a VPN?".

---

## 3. Investigação por IP de origem

**Pergunta:** qual usuário está associado a um IP de origem específico?

**Camada OSI:** 3 (Rede) — o IP é o dado de origem da busca.

```spl
index=VPN_Logs
| spath
| search Source_ip="<IP>"
| stats values(UserName) as UserName count
```

- `search Source_ip="<IP>"` — filtra pelos eventos daquele endereço
- `stats values(UserName) as UserName count` — em vez de só contar, lista os valores distintos de `UserName` encontrados (útil quando mais de um usuário pode compartilhar o mesmo IP) e ainda retorna a contagem

**Resultado:** o IP estava associado a um único nome de usuário específico, respondendo diretamente "quem usou esse endereço?" — pergunta comum em uma investigação de incidente, quando se tem um IP suspeito e se precisa identificar a conta por trás dele.

---

## 4. Investigação por país de origem

**Pergunta:** quantos eventos vieram de fora de um país específico?

**Camada OSI:** 3 (Rede) — a geolocalização deriva do IP de origem.

```spl
index=VPN_Logs
| spath
| search NOT Source_Country="França"
| stats count
```

- `NOT Source_Country="França"` — inverte o filtro, trazendo tudo que **não** bate com essa condição
- `stats count` — conta o total

**Resultado:** 2.814 eventos originados fora da França, de um total de 2.862 — ou seja, a maior parte do tráfego vinha de outros países, o que por si só já seria um dado relevante para investigar se o padrão esperado da empresa fosse concentrado num único país.

---

## 5. Investigação por IP + contagem direta

**Pergunta:** quantos eventos de VPN vieram de um IP específico?

```spl
index=VPN_Logs
| spath
| search Source_ip="<IP>"
| stats count
```

Mesma lógica da seção 3, mas aqui o objetivo é só o volume de eventos daquele IP, sem precisar identificar o usuário — útil, por exemplo, pra medir se um IP está gerando tráfego muito acima do normal (possível indício de automação ou ataque de força bruta).

---

## Conceitos-chave

- **SIEM (Security Information and Event Management)** — sistema que centraliza logs de fontes diferentes, correlaciona eventos e permite investigação e geração de alertas.
- **Índice (index)** — unidade de armazenamento no Splunk onde os dados ficam organizados após a ingestão.
- **SPL (Search Processing Language)** — linguagem de busca do Splunk, estruturada em comandos encadeados por `|`, de forma parecida com um pipeline de shell Linux.
- **`spath`** — comando que interpreta e extrai campos de dados estruturados (como JSON) em cada evento.

## Ferramentas utilizadas

| Ferramenta | Função |
|---|---|
| Splunk | Ingestão, indexação e investigação de logs via SPL |

## Lições aprendidas

- Boa parte do trabalho de um analista de SOC júnior é traduzir uma pergunta de negócio ("quem acessou X?", "quanto tráfego veio de Y?") em uma consulta SPL precisa.
- Dados estruturados (como JSON) exigem um passo extra de extração (`spath`) antes de poderem ser filtrados por campo — um detalhe fácil de esquecer e que trava a investigação se não for feito.
- Escolher o intervalo de tempo certo na busca é um erro comum: dados importados de uma vez só podem cair fora da janela padrão de busca, dando a falsa impressão de que a ingestão falhou.
