# 🚀 Beszel — Instalação via Docker Compose

Beszel é um painel leve de monitoramento de servidores e containers Docker.  
Este repositório fornece um `docker-compose.yml` pronto para uso com rede externa (`app_network`), compatível com Watchtower e proxy reverso Nginx / Nginx Proxy Manager.

---

## 📋 Pré-requisitos

- Docker e Docker Compose instalados
- Rede externa `app_network` criada:
  ```bash
  docker network create traefik_public
  ```
- Arquivo `.env` configurado (veja abaixo)

---

## ⚙️ Configuração do `.env`

Crie um arquivo `.env` na mesma pasta do `docker-compose.yml`:

```env
DOMINIO=beszel.seudominio.com
BESZEL_TOKEN=seu_token_aqui
BESZEL_KEY=sua_chave_aqui
```

> **BESZEL_TOKEN** e **BESZEL_KEY** são gerados na interface do Beszel após o primeiro acesso (veja seção abaixo).

---

## ▶️ Subindo os serviços

```bash
docker compose up -d
```

---

## 🌐 Primeiro acesso e geração do Token/Key

1. **Acesse o painel pelo IP + porta** do servidor enquanto ainda não tem proxy reverso configurado:
   ```
   http://IP_DO_SERVIDOR:8090
   ```
2. Crie sua conta de administrador no primeiro acesso.
3. Vá em **Settings → Agents** e clique em **Add Server**.
4. O Beszel irá gerar automaticamente o **TOKEN** e a **KEY** para o agente.
5. Copie esses valores para o seu `.env` e reinicie os containers:
   ```bash
   docker compose down && docker compose up -d
   ```

---

## 🖥️ Adicionando o servidor local ao painel (agente no mesmo host)

Quando o `beszel-agent` roda no **mesmo servidor** que o `beszel` (dashboard), não é necessário usar IP e porta para a conexão — use o socket Unix compartilhado:

- **Endereço do agente:** `/beszel_socket/beszel.sock`
- **Porta:** deixe em branco (não se aplica para socket)

Isso evita exposição de porta e é mais seguro para comunicação local.

---

## 💾 Identificando o Filesystem correto

O agente precisa saber qual filesystem monitorar. Para descobrir o nome correto no seu servidor:

```bash
lsblk -o NAME,MOUNTPOINT
```

Exemplo de saída:
```
NAME   MOUNTPOINT
sda
└─sda1 /
sdb
└─sdb1 /mnt/dados
```

No `docker-compose.yml`, ajuste a variável `FILESYSTEM` conforme o resultado:
```yaml
FILESYSTEM: "sda1"   # troque pelo seu
```

---

## 🔁 Proxy Reverso

### Nginx — Configuração completa

> ⚠️ **Ponto crítico:** o Beszel usa **SSE (Server-Sent Events)** para atualização em tempo real do painel. Sem a configuração abaixo, o painel **trava** e não atualiza.

```nginx
server {
    listen 443 ssl;
    server_name beszel.seudominio.com;

    # ... suas configs de SSL aqui ...

    location / {
        proxy_pass         http://127.0.0.1:8090;
        proxy_http_version 1.1;

        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;

        # ⚠️ OBRIGATÓRIO para SSE funcionar corretamente
        proxy_set_header   Connection        "";
        proxy_buffering    off;
        gzip               off;
    }
}
```

---

### Nginx Proxy Manager — Configuração

No Nginx Proxy Manager, ao criar o proxy host para o Beszel, vá na aba **Advanced** e insira:

```nginx
proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_buffering off;
gzip off;
```

> Sem isso, o SSE não funciona e o painel trava na atualização em tempo real.

---

## 🏷️ Labels úteis

| Label | Descrição |
|---|---|
| `com.centurylinklabs.watchtower.enable: "true"` | Atualização automática via Watchtower |

Suporte a **Traefik v3** está comentado no `docker-compose.yml`, basta descomentar e ajustar o domínio caso prefira usar o Traefik ao invés do Nginx.

---

## 🔐 OAuth2 / SSO

O Beszel suporta login via OAuth2 usando o PocketBase internamente.

### Provedores suportados

**Externos:** Apple, Discord, GitHub, Google, GitLab, Microsoft, Twitch, Twitter e outros.

**Self-hosted (OIDC):** Authelia, authentik, Gitea, GitLab, Keycloak, Kanidm, mailcow, Pocket ID, ZITADEL.

---

### Como configurar

1. Acesse a interface admin do PocketBase:
   ```
   http://IP_DO_SERVIDOR:8090/_/#/settings
   ```
2. **Desative** a opção *"Hide collection create and edit controls"*
3. Vá em **Collections → users → Options**
4. Ative **OAuth2** e adicione o provedor desejado
5. Reative a opção de ocultar controles

**URL de callback** para registrar no seu provedor OAuth:
```
https://beszel.seudominio.com/api/oauth2-redirect
```

---

### Variáveis de ambiente opcionais

Já estão comentadas no `docker-compose.yml`, basta descomentar conforme necessário:

| Variável | Efeito |
|---|---|
| `USER_CREATION=true` | Cria o usuário automaticamente no primeiro login OAuth |
| `DISABLE_PASSWORD_AUTH=true` | Desativa login por senha — força uso exclusivo de OAuth |

> ⚠️ Não altere `DISABLE_PASSWORD_AUTH` diretamente pelo PocketBase, pois o valor será sobrescrito pelo da variável de ambiente a cada reinicialização.

---

## 📦 Volumes criados

| Volume | Uso |
|---|---|
| `beszel_data` | Dados do dashboard (banco de dados, configs) |
| `beszel_agent_data` | Dados do agente |
| `beszel_socket` | Socket Unix compartilhado entre dashboard e agente |
