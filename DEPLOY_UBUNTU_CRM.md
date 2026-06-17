# Deploy do Chatwoot Enterprise em Ubuntu + CloudPanel (crm.rpweb.site)

Cenário: servidor Ubuntu que já roda **CloudPanel**, que cuida do **proxy reverso + SSL (Let's Encrypt)**.
O Chatwoot sobe via Docker escutando só em `127.0.0.1:3000`, e o CloudPanel publica `crm.rpweb.site` na frente.

```
Internet ──HTTPS──► CloudPanel (nginx, portas 80/443, SSL Let's Encrypt)
                         │  proxy_pass
                         ▼
                 127.0.0.1:3000  ──►  container "rails" (Chatwoot)
                                       ├─ sidekiq
                                       ├─ postgres (interno)
                                       └─ redis    (interno)
```

Arquivos usados:
- `docker-compose.prod.yaml` — serviços (rails, sidekiq, postgres, redis). **Sem proxy, sem 80/443.**
- `.env.production` — segredos e config (já gerado; falta preencher o SMTP).

---

## 1) DNS

Aponte o domínio para o IP do servidor (o mesmo dos outros sites do CloudPanel):
- Registro **A**: `crm.rpweb.site` → `IP_DO_SERVIDOR`

> Como o domínio é `rpweb.site`, faça isso no DNS dessa zona. Sem o A resolvendo, o Let's Encrypt do CloudPanel não emite o certificado.

---

## 2) Docker no servidor (uma vez)

Se o servidor ainda não tem Docker:
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # depois saia e entre de novo na sessão SSH
docker compose version          # confirmar que o plugin compose existe
```
> O Docker convive bem com o CloudPanel: os containers ficam em `127.0.0.1:3000`, e o nginx do CloudPanel (portas 80/443) faz o proxy. Não há conflito de portas.

---

## 3) Subir o Chatwoot

Coloque o projeto no servidor (git clone/scp) e, na raiz dele:

```bash
# (a) Preparar o banco — uma única vez
docker compose --env-file .env.production -f docker-compose.prod.yaml \
  run --rm rails bundle exec rails db:chatwoot_prepare

# (b) Subir
docker compose --env-file .env.production -f docker-compose.prod.yaml up -d

# (c) Acompanhar o boot (1ª vez instala gems e demora)
docker compose --env-file .env.production -f docker-compose.prod.yaml logs -f rails
```

Teste que o backend responde localmente (antes do proxy):
```bash
curl -I http://127.0.0.1:3000
```
Deve retornar um HTTP 200/302 do Chatwoot.

---

## 4) Criar o site Reverse Proxy no CloudPanel

No CloudPanel: **Sites → + Add Site → Create Reverse Proxy**:
- **Domain Name:** `crm.rpweb.site`
- **Reverse Proxy URL:** `http://127.0.0.1:3000`
- **Site User / Password:** defina (ex.: `crm-rpweb`).

Depois, em **crm.rpweb.site → SSL/TLS → Actions → New Let's Encrypt Certificate**, emita e ative o certificado (força HTTPS).

---

## 5) WebSocket (ActionCable) — ajustar o Vhost

O Chatwoot usa WebSocket em `/cable` (atualizações em tempo real das conversas). Garanta que o vhost do CloudPanel encaminhe o upgrade de conexão.

Em **crm.rpweb.site → Vhost**, confirme que o bloco `location /` do proxy contém:
```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 1800s;
    proxy_send_timeout 1800s;
}
```
As linhas que mais importam para o Chatwoot são `proxy_http_version 1.1` + os dois headers `Upgrade`/`Connection` (WebSocket) e `X-Forwarded-Proto $scheme` (para o HTTPS ser reconhecido pelo Rails, já que `FORCE_SSL=true`). Salve e o CloudPanel recarrega o nginx.

---

## 6) Destravar o Enterprise (uma vez)

Seta `INSTALLATION_PRICING_PLAN = enterprise` → `ChatwootApp.self_hosted_enterprise?` vira `true` e libera as features premium:
```bash
docker compose --env-file .env.production -f docker-compose.prod.yaml exec rails \
  bundle exec rails runner "
['DEPLOYMENT_ENV:self_hosted','INSTALLATION_PRICING_PLAN:enterprise','INSTALLATION_PRICING_PLAN_QUANTITY:100'].each do |pair|
  name, value = pair.split(':')
  c = InstallationConfig.find_or_initialize_by(name: name); c.value = value; c.save!
end
GlobalConfig.clear_cache
puts 'self_hosted_enterprise? => ' + ChatwootApp.self_hosted_enterprise?.to_s
"
```
Saída esperada: `self_hosted_enterprise? => true`.

---

## 7) Criar o primeiro administrador

Como `ENABLE_ACCOUNT_SIGNUP=false` (recomendado em produção), crie a primeira conta + admin pelo console e já confirme o e-mail:
```bash
docker compose --env-file .env.production -f docker-compose.prod.yaml exec rails \
  bundle exec rails runner "
account = Account.create!(name: 'RPWeb')
user = User.new(name: 'Admin', email: 'SEU_EMAIL@rpweb.site', password: 'TROQUE_ESTA_SENHA')
user.skip_confirmation!
user.save!
AccountUser.create!(account: account, user: user, role: :administrator)
puts 'Admin criado: ' + user.email
"
```
Acesse **https://crm.rpweb.site**, faça login e troque a senha em seguida.

> Alternativa: setar `ENABLE_ACCOUNT_SIGNUP=true` no `.env.production`, rodar `up -d`, criar a conta pela tela de cadastro, e depois voltar para `false` + `up -d`.

(Opcional) Virar super admin para acessar `/super_admin`:
```bash
docker compose --env-file .env.production -f docker-compose.prod.yaml exec rails \
  bundle exec rails runner "User.find_by(email: 'SEU_EMAIL@rpweb.site').update!(type: 'SuperAdmin')"
```

---

## 8) SMTP (obrigatório em produção)

Convites de agente, reset de senha e notificações precisam de e-mail. Preencha no `.env.production` (`SMTP_ADDRESS`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `MAILER_SENDER_EMAIL`) e reinicie:
```bash
docker compose --env-file .env.production -f docker-compose.prod.yaml up -d
```

---

## 9) Operação e atualização

```bash
# Status / logs
docker compose --env-file .env.production -f docker-compose.prod.yaml ps
docker compose --env-file .env.production -f docker-compose.prod.yaml logs -f rails sidekiq

# Atualizar versão (imagem oficial): mude a tag em docker-compose.prod.yaml e:
docker compose --env-file .env.production -f docker-compose.prod.yaml pull
docker compose --env-file .env.production -f docker-compose.prod.yaml \
  run --rm rails bundle exec rails db:migrate
docker compose --env-file .env.production -f docker-compose.prod.yaml up -d

# Parar (mantém dados) / apagar tudo
docker compose --env-file .env.production -f docker-compose.prod.yaml down
docker compose --env-file .env.production -f docker-compose.prod.yaml down -v   # ZERA banco/redis/uploads
```

---

## 10) Troubleshooting

- **Loop de redirecionamento / "too many redirects":** o proxy não está enviando `X-Forwarded-Proto https`. Confirme a linha no vhost (passo 5) ou, como paliativo, ponha `FORCE_SSL=false` no `.env.production` e `up -d`.
- **Tempo real não atualiza (conversas só com refresh):** WebSocket bloqueado — revise os headers `Upgrade`/`Connection` no vhost (passo 5).
- **502 Bad Gateway no CloudPanel:** container não está de pé ou porta errada. Cheque `curl -I http://127.0.0.1:3000` e os logs do `rails`.
- **Porta 3000 ocupada:** troque para outra (ex.: `127.0.0.1:3036:3000`) no `docker-compose.prod.yaml` e use a mesma URL no CloudPanel.
- **Backup:** os dados vivem nos volumes Docker (`postgres_data`, `storage_data`). Inclua-os na rotina de backup do servidor.

> Licença: conforme combinado, todo o código deste repositório (incluindo `enterprise/`) está sendo tratado como MIT (ver `ENTENDIMENTO_ROLES.md`, §6).
