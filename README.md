# HeroGuard — Stack Docker

Stack da aplicação HeroGuard em Docker: site estático (nginx) + **API de
cadastro** (Node + pg) + **PostgreSQL** local, com integração com o
**Traefik** já existente na VPS, atendendo **heroguard.my** (mesma
infraestrutura das outras stacks da VPS, como o inlaudo).

**Status:** a stack anterior (`chatguard`, domínio `chatguard.life`) entrou em
produção em 2026-10-19. Em 2026-10-23 todo o projeto foi renomeado para
**HeroGuard** — domínio `heroguard.my` (alterado de `heroguard.life` em
2026-10-23), imagens `wallesonnn/heroguard-app` e `wallesonnn/heroguard-api`,
containers `heroguard-app`/`heroguard-api`/`heroguard-db`, routers/services
do Traefik `heroguard` e banco `heroguard`.

> **Stack renomeada no ar desde 2026-10-23:** `https://heroguard.my` e
> `https://www.heroguard.my` respondem com HTTP 200 e certificado Let's Encrypt
> válido (SAN para os dois hosts). O DNS de ambos aponta para a VPS — o Traefik
> só emite o certificado quando o domínio resolve.

## Serviços

| Serviço | Container | Papel |
|---|---|---|
| `app` | `heroguard-app` | nginx servindo o site estático; faz proxy de `/api/` para `api:3000` |
| `api` | `heroguard-api` | Node + pg; recebe `POST /api/waitlist` e grava na tabela `waitlist`; cria o schema na inicialização |
| `db` | `heroguard-db` | Postgres 17 (volume `pgdata`); sem porta no host |

A `api` só fica na rede interna (`default`): o único ponto público é o nginx.
O `db` também entra na rede `web` com o alias **`heroguard-db`** para que o
n8n (mesma VPS, mesma rede) consulte a tabela `waitlist` sem porta exposta.

## Estrutura

```text
heroguard-docker/
├── Dockerfile               # build do site (Next.js → nginx servindo out/)
├── Dockerfile.dockerignore  # exclusões do contexto DESTE Dockerfile (o nome importa!)
├── docker-compose.yml       # serviços app + api + db + labels do Traefik
├── nginx.conf               # estático + proxy /api/ (com resolver do Docker)
├── .env.example             # configuração local (referência; o real não sobe)
└── README.md                # este arquivo
```

> No GitHub, este repositório versiona **apenas** `README.md` e
> `docker-compose.yml`. Os demais arquivos são locais (necessários ao build)
> e ficam fora do versionamento, ignorados pelo `.gitignore` da pasta.

O código da API fica em **`api/` no repo do app** (`heroguard-app/api/`),
junto do `build.sh`, que constrói e publica as duas imagens no Docker Hub.
O Dockerfile da API é `api/Dockerfile`, com contexto `api/`.

## Pré-requisitos

- VPS com Docker e Docker Compose.
- Traefik já rodando na VPS com:
  - rede externa **`web`** (hardcoded no compose, como no inlaudo);
  - entrypoint HTTPS **`websecure`**;
  - resolutor de certificados **`lets-encrypt`**.
- DNS: `heroguard.my` apontando para o IP da VPS.
- Para incluir o vídeo: coloque o arquivo em `public/videos/` **antes** do
  build (o vídeo entra na imagem, não sobe para lugar nenhum).

> A regra do Traefik já cobre `heroguard.my` **e** `www.heroguard.my**
> (o DNS do `www` aponta para a VPS; o certificado é emitido com SAN para
> os dois hosts).

## Deploy no Portainer

**Opção 1 — Web editor (colar):**

1. **Stacks → Add stack** — nome da stack: `heroguard`.
2. **Build method: Web editor** — cole o conteúdo de `docker-compose.yml`.

**Opção 2 — Repository (este repo no GitHub):**

1. **Stacks → Add stack** — nome da stack: `heroguard`.
2. **Build method: Repository** — aponte para
   `https://github.com/wallesonn/heroguard-docker`, branch `main`, compose
   path `docker-compose.yml`.

**Em ambos os casos:**

3. Em **Environment variables**, defina no mínimo:
   - `POSTGRES_PASSWORD` — senha forte (troque a padrão; **evite `@ : / # ?`**,
     que quebram o parse da `DATABASE_URL` montada no compose);
   - `DOCKER_IMAGE` — opcional, para travar a versão do site
     (ex.: `wallesonnn/heroguard-app:v0.1.6`; padrão: `latest`);
   - `API_IMAGE` — opcional, para travar a versão da API
     (ex.: `wallesonnn/heroguard-api:v0.1.6`; padrão: `latest`).
4. **Deploy the stack**. Depois de alguns segundos, `https://heroguard.my`
   deve responder; o certificado é emitido pelo Traefik.

> O compose não tem bloco `build:` de propósito: as imagens são sempre
> puxadas prontas do Docker Hub (publicadas via `./build.sh`). Um bloco
> `build:` falha no Portainer, porque o repo não contém o código-fonte do
> app. Para atualizar a produção depois de mudanças no repo, use
> **Update the stack**.

## Alternativa: CLI na VPS

```bash
cd heroguard-docker
cp .env.example .env      # troque POSTGRES_PASSWORD
docker compose pull
docker compose up -d
docker compose ps
```

## Build e atualização das imagens

```bash
# na sua máquina (o vídeo de public/videos/ entra na imagem do site)
./build.sh vX.Y.Z         # publica heroguard-app E heroguard-api no Docker Hub
```

Depois, no Portainer: **Pull and redeploy** na stack (seguindo `latest`) ou
troque `DOCKER_IMAGE`/`API_IMAGE` para a versão exata e faça
**Update the stack**.

## Banco de dados (Postgres)

- Banco criado automaticamente na primeira subida com as variáveis
  (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`).
- A tabela **`waitlist`** é criada pela própria API na inicialização
  (idempotente; não há passo manual de migração).
- Dados persistidos no volume `pgdata` (sobrevive a `down`/`up`; apague apenas
  de propósito, com `docker compose down -v`).
- **Sem porta publicada no host** — o bloco `ports:` do `db` está desabilitado
  no compose porque a VPS já tem outro Postgres ocupando a 5432. O acesso é
  pelas redes Docker: host `db` (serviços da stack) e alias `heroguard-db`
  (n8n, na rede `web`).
- Consulta rápida (sem senha, socket local dentro do container):

  ```bash
  docker exec heroguard-db psql -U heroguard -d heroguard \
    -c "SELECT id, name, email, phone, status, created_at FROM waitlist ORDER BY id DESC LIMIT 10;"
  ```

- Consumo pelo n8n (nó Postgres): host `heroguard-db`, porta `5432`,
  database/usuário do env, senha `POSTGRES_PASSWORD`. Sugestão de fluxo:
  `SELECT ... FROM waitlist WHERE status = 'novo' ORDER BY created_at`;
  após contatar o lead, marcar `status = 'contatado'` e
  `updated_at = now()`.

## Segurança

- Troque `POSTGRES_PASSWORD` antes do primeiro deploy.
- O `.env` real nunca é versionado; o `Dockerfile.dockerignore` impede que
  segredos e artefatos locais entrem no contexto de build.
- O Postgres não publica porta no host — alcançável apenas pelas redes
  Docker (`default` e `web`).
- ⚠️ **`POSTGRES_USER`/`POSTGRES_PASSWORD` só valem na primeira criação do
  volume.** Trocar a variável depois NÃO troca a senha do banco: alinhe com
  `ALTER USER` dentro do container ou recrie o volume (ver lições abaixo).

## Lições do primeiro deploy (2026-10-19)

Problemas encontrados e resolvidos — úteis para os próximos deploys/domínios:

- **Bloco `build:` no compose quebra o Portainer** — ao fazer deploy pelo
  método *Repository*, o Portainer clona só este repo (sem o código-fonte
  do app) e falha em `lstat .../heroguard-docker`. Por isso o compose não tem
  `build:`: a imagem é sempre puxada pronta do Docker Hub.
- **DNS precisa propagar ANTES do deploy** — o Let's Encrypt falhou com
  `no valid A records found` porque o DNS do domínio era novo. Após
  5 validações falhas, o LE aplicou rate limit de 1h
  (`too many failed authorizations`). O Traefik retenta sozinho e o
  certificado saiu quando o limite expirou — mas evite o ciclo: confirme
  `dig +short dominio` retornando o IP da VPS antes de subir a stack.
- **Porta 5432 do host ocupada** — outro Postgres da VPS já a usa; o
  mapeamento de porta do `db` ficou desabilitado (ver seção Banco de dados).
- **`traefik.enable=false` nos serviços não-públicos** — o Traefik da VPS usa
  `exposedByDefault`, então tentava rotear o Postgres e logava
  `unable to find the IP address` a cada scan. A label desliga isso; aplique
  em qualquer serviço não-público (`db` e `api` têm).

## Lições do deploy da API (2026-10-28)

- **`28P01` — senha divergente entre env e volume:** a API ficou em loop
  `banco indisponível` até a senha do banco ser alinhada à do stack com
  `docker exec -it heroguard-db psql -U heroguard -d heroguard -c "ALTER
  USER heroguard WITH PASSWORD '...';"` e `docker restart heroguard-api`.
  Causa: o volume já existia com a senha antiga. Na stack de teste o caso
  foi um `POSTGRES_USER` herdado de um volume antigo — resolvido recriando o
  volume (`docker volume rm`), aceitável porque não havia dado algum.
- **nginx + `proxy_pass` com nome de serviço:** sem `resolver 127.0.0.11` +
  variável no `proxy_pass`, o nginx resolve `api:3000` no startup e **morre**
  com `host not found in upstream` se o container da API ainda não existir
  (restart loop). Com o resolver, a resolução acontece a cada requisição.
- **O nome do dockerignore importa:** com
  `docker build -f heroguard-docker/Dockerfile .`, o BuildKit só aplica
  `heroguard-docker/Dockerfile.dockerignore` (ou um `.dockerignore` na raiz
  do contexto). O arquivo chamava-se apenas `.dockerignore` dentro da pasta
  e **nunca era aplicado** — `node_modules`, `.next` e `out` entravam no
  contexto do build do site. Foi renomeado para `Dockerfile.dockerignore`.
- **`psql` local sem senha:** dentro do container do Postgres, o socket
  local é `trust` (`local all all trust`) — consultas administrativas não
  precisam de senha; conexões TCP exigem as credenciais do env.
