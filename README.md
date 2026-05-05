<div align="center">

# neko lab

sandbox de navegação isolado, auto-hospedado em vps.

</div>

---

um lab pessoal que roda firefox e chromium completos num servidor remoto e transmite a tela via webrtc. você acessa pelo navegador, controla com mouse e teclado, e toda a navegação acontece isolada do seu sistema e da sua rede pessoal.

bom pra:

- testar extensões e sites suspeitos sem expor o navegador principal
- separar contas e sessões (banco, trabalho, pessoal) sem perfis cruzados
- ter um navegador "descartável" sempre limpo
- estudar self-hosting, containers e infraestrutura

## stack

| camada | tecnologia |
|---|---|
| navegadores virtuais | [neko](https://github.com/m1k1o/neko) (firefox + chromium) |
| reverse proxy | [traefik](https://traefik.io/) v2.11 |
| tls | let's encrypt (automático) |
| dns | [duckdns](https://www.duckdns.org/) (gratuito) |
| orquestração | docker compose |
| frontend | html / css / js puro, sem build, sem dependências |

## arquitetura

```
                    você
                     │ https
                     ▼
         your-lab.duckdns.org
                     │
              ┌──────┴──────┐
              │   traefik   │  reverse proxy + tls
              └──────┬──────┘
        ┌────────────┼────────────┐
        ▼            ▼            ▼
     launcher   neko firefox  neko chromium
     (nginx)    (firefox vm)  (chromium vm)
```

cada navegador roda em seu próprio container isolado. o traefik distribui o tráfego baseado no subdomínio:

- `your-lab.duckdns.org` → launcher (página inicial)
- `firefox.your-lab.duckdns.org` → sessão firefox
- `chrome.your-lab.duckdns.org` → sessão chromium

## requisitos

- vps com ubuntu 22.04 ou 24.04
- mínimo 4 vcpu / 8 gb ram (1 navegador) — recomendado 6 vcpu / 12 gb (2 simultâneos)
- portas liberadas: `80` e `443` tcp + `52100-52400` udp (webrtc)
- subdomínio no duckdns ou domínio próprio

## deploy

### 1. clona o repo no servidor

```bash
git clone https://github.com/dgolaus/neko-lab.git
cd neko-lab
```

### 2. configura o `.env`

```bash
cp .env.example .env
nano .env
```

preenche:

| variável | descrição |
|---|---|
| `DOMAIN` | seu domínio (ex: `your-lab.duckdns.org`) |
| `PUBLIC_IP` | ip público do vps (`curl -4 ifconfig.io`) |
| `LETSENCRYPT_EMAIL` | e-mail para avisos do let's encrypt |
| `NEKO_ADMIN_PASSWORD` | senha de admin (controle total) |
| `NEKO_USER_PASSWORD` | senha de usuário (apenas visualização) |

dica pra gerar senhas fortes:

```bash
openssl rand -base64 24
```

### 3. cria a estrutura de diretórios

```bash
mkdir -p acme profiles/firefox profiles/chrome
chown -R 1000:1000 profiles/
touch acme/acme.json && chmod 600 acme/acme.json
```

### 4. firewall

```bash
ufw allow 22/tcp                # ssh
ufw allow 80/tcp                # http
ufw allow 443/tcp               # https
ufw allow 52100:52200/udp       # webrtc firefox
ufw allow 52300:52400/udp       # webrtc chromium
ufw --force enable
```

### 5. sobe os containers

```bash
docker compose pull
docker compose up -d
```

depois de uns 30-60 segundos, acessa `https://your-lab.duckdns.org` e faz login com as credenciais que você definiu no `.env`.

## customização visual

todo o estilo fica em `frontend/index.html`. as variáveis css estão no topo do `<style>`:

```css
:root {
  --bg: #000;
  --purple: #a855f7;
  --purple-bright: #c084fc;
  --text: #ede9fe;
  --muted: #a78bfa;
}
```

troca a cor base e tudo se ajusta. atualmente a paleta é preto + roxo, minimalista.

## comandos úteis

```bash
# status dos containers
docker compose ps

# logs do traefik (cert issuance, requests)
docker compose logs -f traefik

# logs do neko firefox
docker compose logs -f neko-firefox

# reinicia um navegador (limpa estado de runtime)
docker compose restart neko-firefox

# para tudo (sem apagar dados)
docker compose stop

# liga de novo
docker compose start

# atualiza imagens pra última versão
docker compose pull && docker compose up -d
```

## persistência de perfis

os volumes `./profiles/firefox` e `./profiles/chrome` mantêm cookies, histórico, extensões e bookmarks entre reinícios. pra resetar um perfil:

```bash
docker compose stop neko-firefox
sudo rm -rf profiles/firefox/*
docker compose start neko-firefox
```

## keep-alive do duckdns

duckdns desativa subdomínios sem ping por 30 dias. configura um cron pra manter o domínio vivo:

```bash
# pega seu token em duckdns.org → painel
TOKEN="seu-token-aqui"

(crontab -l 2>/dev/null; echo "0 6 * * * curl -k 'https://www.duckdns.org/update?domains=your-lab&token=$TOKEN&ip=' >/dev/null 2>&1") | crontab -
```

## custo aproximado

| item | valor |
|---|---|
| vps (contabo cloud vps 20, us-east) | ~r$ 67/mês |
| domínio duckdns | grátis |
| domínio próprio (opcional) | ~r$ 60/ano |

## troubleshooting

**tela preta após login** → webrtc não conectou. verifica se as portas udp `52100-52400` estão abertas no firewall do vps e do provedor (alguns cloud providers têm firewall extra além do ufw).

**aviso "não seguro" no navegador** → cert let's encrypt falhou. tenta forçar re-emissão:

```bash
docker compose stop traefik
rm acme/acme.json && touch acme/acme.json && chmod 600 acme/acme.json
docker compose up -d traefik
sleep 60
docker compose logs traefik | grep -i obtained
```

se persistir, troca o desafio do let's encrypt no `docker-compose.yaml` de `tlschallenge=true` para `httpchallenge=true` + `httpchallenge.entrypoint=web`.

**sessão lenta / travando** → vps pode estar com pouca ram. verifica com `free -h` e `docker stats`. considera upgrade para o cloud vps 30 (24 gb ram) se for usar com várias pessoas.

## referências

- [neko](https://neko.m1k1o.net/) — projeto original do navegador virtual
- [traefik docs](https://doc.traefik.io/traefik/) — reverse proxy
- [duckdns](https://www.duckdns.org/) — dns dinâmico gratuito
- [docker compose](https://docs.docker.com/compose/) — orquestração

## licença

mit
