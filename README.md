# Causa raiz — redirect indevido allin.com.br / blog.allin.com.br → wake.tech

**Data da investigação:** 2026-10-07
**Servidor:** EC2 `i-02503e546ac19e850` (`wake-prod-site`), IP privado `10.72.11.42`, IP público `44.221.206.86`
**Docroot:** `/var/www/funcionalidades`

## Resumo

`allin.com.br` e `blog.allin.com.br` retornam 301 para `https://wake.tech/`. A causa **não é o Apache** e **não é um plugin/malware** — é a própria instalação do WordPress, que tem as options `siteurl` e `home` apontando para `wake.tech` em vez do domínio próprio.

## Causa raiz

```
siteurl = https://wake.tech/funcionalidades
home    = https://wake.tech/funcionalidades
```

O WordPress usa essas duas options no redirect canônico (`redirect_canonical()` → `wp_redirect()`). Qualquer request feito em `allin.com.br` ou `blog.allin.com.br` não bate com o `home` configurado, e o WordPress redireciona (301) para `wake.tech`.

Não é multisite — `wp core is-installed --network` confirmou que é uma **única instalação** respondendo pelos dois domínios. Logo, **uma correção resolve os dois redirects**.

## Evidência coletada

**1. Header do redirect (confirma que é o WordPress, não o Apache/CDN):**
```
HTTP/1.1 301 Moved Permanently
Server: Apache/2.4.52 (Ubuntu)
X-Redirect-By: WordPress
Location: https://wake.tech/
```
`X-Redirect-By: WordPress` só é adicionado pela função `wp_redirect()` do core do WP.

**2. Apache já estava corrigido (não é mais a causa):**
Nos vhosts `/etc/apache2/sites-enabled/allin.com.br.conf` e `blog.allin.com.br.conf`, a regra antiga de redirect está **comentada**:
```apache
#     RewriteRule ^.*$ https://wake.tech/ [R=308,L]
```

**3. Options do WordPress (mesma instalação, confirmado para os dois domínios):**
```bash
$ wp option get siteurl --path=/var/www/funcionalidades --url=https://allin.com.br
https://wake.tech/funcionalidades
$ wp option get siteurl --path=/var/www/funcionalidades --url=https://blog.allin.com.br
https://wake.tech/funcionalidades
$ wp core is-installed --network --path=/var/www/funcionalidades
Error: This is not a multisite installation.
```

**4. DNS:** `allin.com.br` e `blog.allin.com.br` resolvem para o mesmo IP (`44.221.206.86`).

## Descartado durante a investigação

- **mu-plugin `wake-ajuste-telefone.php`**: não tem relação com o redirect — é só CSS de padding de um campo de telefone (intl-tel-input), com comentário mencionando `marketing.wake.tech` que levou a investigar essa pista, mas o conteúdo do arquivo não mexe em redirect/URL.
- **Apache / .htaccess**: regra de redirect já estava comentada antes desta investigação.
- **Malware/comprometimento no WordPress** (hipótese levantada inicialmente via ChatGPT): não encontrado nenhum indício (`base64_decode`, `eval(`, código injetado) — os únicos `wp_redirect()` encontrados no grep são usos legítimos de plugins (Rank Math, WP Rocket, ACF, All-In-One Security, etc.), sem relação com `wake.tech`.

## Correção proposta

```bash
# Backup dos valores atuais antes de qualquer mudança
sudo -u www-data wp option get siteurl --path=/var/www/funcionalidades > /root/siteurl.backup.txt
sudo -u www-data wp option get home --path=/var/www/funcionalidades > /root/home.backup.txt

# Correção
sudo -u www-data wp option update siteurl 'https://allin.com.br' --path=/var/www/funcionalidades
sudo -u www-data wp option update home 'https://allin.com.br' --path=/var/www/funcionalidades
```

> **Atenção:** confirmar com o time se `blog.allin.com.br` deve ter uma `home` própria (ex.: subdiretório `/blog`) ou se compartilha a mesma instalação sem distinção — hoje ambos resolvem para a mesma option porque não é multisite.

## Validação pós-fix

```bash
curl -sIL https://allin.com.br/ | head -20
curl -sIL https://blog.allin.com.br/ | head -20
```
Esperado: `HTTP/1.1 200 OK`, sem `Location` para `wake.tech`.

## Rollback

Caso algo quebre após a correção, reverter para o valor original:

```bash
sudo -u www-data wp option update siteurl 'https://wake.tech/funcionalidades' --path=/var/www/funcionalidades
sudo -u www-data wp option update home 'https://wake.tech/funcionalidades' --path=/var/www/funcionalidades
```

(ou usar o conteúdo salvo em `/root/siteurl.backup.txt` e `/root/home.backup.txt`, caso tenha sido diferente do valor acima)

## Referências

- Thread Slack: *indexação-indevida-redirect-allin* (04/09)
- Backup AWS Backup feito antes de qualquer alteração: instance `i-02503e546ac19e850`, recovery point `ami-0edfaf54b3a2fe231`, retenção 60 dias
