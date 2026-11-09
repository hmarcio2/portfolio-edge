# portfolio-edge

Borda da VPS da Hostinger: **Caddy** com HTTPS automático (Let's Encrypt) e a rede Docker `edge`, à qual todos os projetos
se conectam. Cada projeto (repositório próprio) instala no deploy o seu trecho de Caddy:

| Pasta na VPS | Quem escreve | Exemplo |
|---|---|---|
| `/opt/edge/routes/*.caddy` | deploy do `portfolio-chat` | `/api/chat/*` no domínio principal |
| `/opt/edge/sites/*.caddy` | deploy de `portfolio-filmes`, `portfolio-planilhas`, `portfolio-langfuse` | `filmes.SEUDOMINIO`, `planilhas.SEUDOMINIO`, `langfuse.SEUDOMINIO` |

O domínio principal serve o `portfolio-site` (container `site`, porta 8080).

## Preparar a VPS (uma vez)
Pressupõe uma VPS Ubuntu da Hostinger com Docker e Compose v2.20 ou superior (há template com Docker no painel).

```bash
# 1. usuário de deploy (sem senha, só chave SSH) com acesso ao Docker
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy
sudo mkdir -p /opt/edge/sites /opt/edge/routes /opt/apps && sudo chown -R deploy:deploy /opt/edge /opt/apps
sudo -u deploy mkdir -p ~deploy/.ssh && sudo -u deploy sh -c 'cat >> ~deploy/.ssh/authorized_keys'   # cole a chave PÚBLICA e Ctrl+D

# 2. firewall: só SSH, HTTP e HTTPS
sudo ufw allow OpenSSH && sudo ufw allow 80,443/tcp && sudo ufw allow 443/udp && sudo ufw enable

# 3. configuração da borda
sudo -u deploy cp /opt/edge/.env.example /opt/edge/.env   # depois de rodar o deploy da borda uma vez, edite DOMAIN
```

Gere a chave de deploy **no seu computador** (`ssh-keygen -t ed25519 -f deploy_key -C github-actions`), instale a pública na VPS
e guarde a privada em GitHub Actions. Segredos do ambiente `production` de **cada** repositório (Settings > Environments):

| Segredo | Valor |
|---|---|
| `VPS_HOST` | IP ou hostname da VPS |
| `VPS_USER` | `deploy` |
| `VPS_SSH_KEY` | conteúdo da chave privada |
| `VPS_KNOWN_HOSTS` | saída de `ssh-keyscan -p 22 SEU_IP` (confira a impressão digital no painel da Hostinger) |
| variável `VPS_PORT` | opcional, se o SSH não estiver na 22 |

## Ordem do primeiro deploy
1. DNS: registros `A` de `SEUDOMINIO`, `filmes.SEUDOMINIO`, `planilhas.SEUDOMINIO` (e `langfuse.SEUDOMINIO`, se hospedar o Langfuse) para o IP da VPS.
2. Este repositório: `Actions > Deploy edge`. Depois, na VPS, crie `/opt/edge/.env` (`DOMAIN=...`) e rode o deploy de novo
   (o primeiro falha de propósito se o `.env` não existir).
3. `portfolio-site`, `portfolio-chat`, `portfolio-filmes`, `portfolio-planilhas`: em cada um, crie `/opt/apps/<nome>/.env`
   a partir do `.env.example` do repositório e rode `Actions > Deploy`.

## Segurança
- Só o Caddy publica portas; os apps ficam na rede interna `edge`.
- Containers dos apps: `read_only`, sem capabilities, `no-new-privileges`, limite de memória.
- O deploy usa um usuário sem senha e sem sudo, chave exclusiva para o CI e `StrictHostKeyChecking=yes`.
- O login no GHCR durante o deploy usa o `GITHUB_TOKEN` do job (expira com ele) e é desfeito no fim.
