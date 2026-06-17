# Rodar o Chatwoot Enterprise (completo) localmente no Mac

Guia para subir esta instância via `docker compose` com **todas as features Enterprise** (custom roles, SLA, Captain/AI, etc.) destravadas.

Arquivos criados para isso:
- `docker-compose.enterprise.yaml` — os serviços (Rails, Sidekiq, Postgres, Redis).
- `.env` — segredos e config (já gerado, com `SECRET_KEY_BASE` e senhas únicas).

---

## Build do código local vs. imagem oficial — a diferença na prática

São dois jeitos de obter a imagem do app. **Para as features Enterprise, os dois se comportam igual** — porque ambos contêm a pasta `enterprise/`, e o que liga o premium é uma config no banco (não a forma de buildar). A diferença real é outra:

| | **Imagem oficial prebuilt** (Opção A, padrão) | **Build do código local** (Opção B) |
|---|---|---|
| O que roda | O código **publicado** da v4.15.1 | **O seu** working copy (este repo, com suas alterações) |
| Tempo p/ subir | Minutos (só baixa a imagem) | 1º build pesado (~10–20 min: gems + assets Vite) |
| Suas modificações em roles aparecem? | **Não** | **Sim** |
| Apple Silicon (M1/M2/M3/M4) | Imagem é **só amd64** → roda **emulada** (mais lenta) | Builda **nativo arm64** (mais rápida) |
| Quando usar | Só quero **ver/avaliar** o Enterprise funcionando | Vou **mexer no código** (ex.: roles) e testar minhas mudanças |

**Recomendação:** como você quer só rodar localmente por enquanto, comece pela **Opção A** (padrão do compose). No momento em que for testar suas próprias mudanças de roles, troque para a **Opção B** (instruções no topo do `docker-compose.enterprise.yaml` — é trocar 2 linhas).

> Observação: a separação "OSS `app/`" vs "Enterprise `enterprise/`" deste projeto é **técnica** (injeção via `prepend_mod_with` + feature flag), não muda o jeito de buildar.

---

## Pré-requisitos

- **Docker Desktop** instalado e rodando.
- Em Apple Silicon, mantenha o Docker Desktop atualizado (a opção **"Use Rosetta for x86/amd64 emulation"** em Settings → General ajuda a Opção A a rodar mais rápido).

---

## Passo a passo (Opção A — imagem oficial)

Rode tudo a partir da raiz do projeto (`/Users/rafael/Projects/chatwoot`).

### 1) Preparar o banco (uma única vez)
Cria o schema e dados iniciais:
```bash
docker compose -f docker-compose.enterprise.yaml run --rm rails bundle exec rails db:chatwoot_prepare
```

### 2) Subir os serviços
```bash
docker compose -f docker-compose.enterprise.yaml up -d
```
Acompanhe os logs (a 1ª subida instala gems faltantes e demora um pouco):
```bash
docker compose -f docker-compose.enterprise.yaml logs -f rails
```

### 3) Abrir e criar a primeira conta
Acesse **http://localhost:3000** e crie a primeira conta (o primeiro usuário vira administrador da conta).

### 4) Destravar o Enterprise (uma única vez)
Isto seta `INSTALLATION_PRICING_PLAN = enterprise`, o que faz `ChatwootApp.self_hosted_enterprise?` virar `true` e libera as features premium:
```bash
docker compose -f docker-compose.enterprise.yaml exec rails bundle exec rails runner "
['DEPLOYMENT_ENV:self_hosted','INSTALLATION_PRICING_PLAN:enterprise','INSTALLATION_PRICING_PLAN_QUANTITY:100'].each do |pair|
  name, value = pair.split(':')
  c = InstallationConfig.find_or_initialize_by(name: name); c.value = value; c.save!
end
GlobalConfig.clear_cache
puts 'self_hosted_enterprise? => ' + ChatwootApp.self_hosted_enterprise?.to_s
"
```
A última linha deve imprimir `self_hosted_enterprise? => true`.

### 5) (Opcional) Virar super admin
Para acessar o painel `/super_admin` (gerencia contas, features, etc.). Troque pelo seu e-mail:
```bash
docker compose -f docker-compose.enterprise.yaml exec rails bundle exec rails runner "
u = User.find_by(email: 'SEU_EMAIL@exemplo.com'); u.update!(type: 'SuperAdmin'); puts 'super admin: ' + u.reload.type.to_s
"
```
Painel: **http://localhost:3000/super_admin**

### 6) Conferir as features premium na conta
Se alguma feature premium não aparecer na UI da conta, habilite-a explicitamente (custom roles, SLA, etc.):
```bash
docker compose -f docker-compose.enterprise.yaml exec rails bundle exec rails runner "
acc = Account.first
acc.enable_features('custom_roles','sla','audit_logs','help_center','captain_integration')
acc.save!
puts acc.enabled_features.keys.inspect
"
```
> Alternativa via UI: `/super_admin` → Accounts → (sua conta) → aba **Features**.

Depois, em **Configurações → Cargos de Usuário (Custom Roles)** você já consegue criar cargos com as permissões granulares (`conversation_manage`, `contact_manage`, `report_manage`, `knowledge_base_manage`, etc.).

---

## Operação do dia a dia

```bash
# Parar (mantém os dados nos volumes)
docker compose -f docker-compose.enterprise.yaml down

# Subir de novo
docker compose -f docker-compose.enterprise.yaml up -d

# Logs
docker compose -f docker-compose.enterprise.yaml logs -f rails sidekiq

# Console Rails
docker compose -f docker-compose.enterprise.yaml exec rails bundle exec rails console

# APAGAR TUDO (zera banco/redis/uploads — recomeça do zero)
docker compose -f docker-compose.enterprise.yaml down -v
```

---

## Trocar para a Opção B (build do código local)

No `docker-compose.enterprise.yaml`, no bloco `x-base`:
1. Comente as linhas `image: chatwoot/chatwoot:v4.15.1` e `platform: linux/amd64`.
2. Descomente o bloco `build:`.

Depois:
```bash
docker compose -f docker-compose.enterprise.yaml build
docker compose -f docker-compose.enterprise.yaml run --rm rails bundle exec rails db:chatwoot_prepare   # se for um banco novo
docker compose -f docker-compose.enterprise.yaml up -d
```

---

## Notas e limitações

- **Apple Silicon + Opção A:** a imagem oficial é só `amd64` (confirmado no Docker Hub) → roda sob emulação. Funciona, mas é mais lenta. Para performance nativa, use a **Opção B**.
- **Não é uma config de produção:** portas estão presas em `127.0.0.1`, sem HTTPS/proxy reverso, signup aberto e e-mail desativado. Serve para avaliação/dev local.
- **Licença:** conforme combinado, todo o código deste repositório (incluindo `enterprise/`) está sendo tratado como MIT (ver `ENTENDIMENTO_ROLES.md`, §6). Rodar as features Enterprise sem licença comercial só é adequado porque é essa a sua decisão sobre este repositório.
- **`SECRET_KEY_BASE`/senhas** no `.env` são exclusivos desta instalação — não reutilize em produção.
