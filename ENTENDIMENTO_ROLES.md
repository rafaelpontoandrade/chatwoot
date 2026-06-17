# Documento de Entendimento — Chatwoot (foco: Roles & Permissões)

> Sessão de **análise read-only**. Nenhum arquivo de código foi editado, criado ou removido.
> Toda afirmação cita o caminho real de um arquivo lido. O que não foi confirmado está marcado como **(não confirmado)**.
> Data: 2026-06-17.

---

## 1. Resumo das diretrizes que vão me restringir (Fase 0)

`CLAUDE.md` é um **symlink para `AGENTS.md`** (mesmo conteúdo). Regras que importam para qualquer alteração:

**Setup / dev / teste / lint**
- Setup: `bundle install && pnpm install`. Dev: `pnpm dev` ou `overmind start -f ./Procfile.dev`.
- Ruby gerido via `rbenv` (versão em `.ruby-version`); antes de `bundle`/`rspec`, rodar `eval "$(rbenv init -)"`. Sempre preferir `bundle exec`.
- Lint Ruby: `bundle exec rubocop -a` (máx. 150 colunas). Lint JS/Vue: `pnpm eslint` / `pnpm eslint:fix` (Airbnb base + Vue 3).
- Teste Ruby: `bundle exec rspec spec/...`. Teste JS: `pnpm test`.

**Padrões de código**
- Vue: **Composition API com `<script setup>`** sempre; componentes em PascalCase; eventos em camelCase.
- **Tailwind only** — proibido CSS custom, scoped CSS ou inline styles. Cores em `tailwind.config.js`.
- I18n obrigatório (sem strings cruas em templates).
- Models: validar presença/unicidade, criar índices apropriados.
- Frontend novo deve usar `components-next/` (o resto está sendo depreciado).

**Filosofia do projeto**
- Foco MVP / happy-path; menor mudança de código possível; sem defensive programming desnecessário.
- Clareza acima de abstração; iterar **após confirmação**.
- **Não escrever specs a menos que explicitamente pedido.**
- Não criar múltiplas versões/backups da mesma lógica; remover código morto.

**Git / PR**
- Conventional Commits: `type(scope): subject`. **Não referenciar Claude** nas mensagens.
- Worktree + branch por tarefa (workflow `.codex/`, `Procfile.worktree`).
- PR começa com parágrafo de produto + seção `Closes` + `How to test` (feature) / `How to reproduce` (bugfix).

**Enterprise Edition (CRÍTICO para roles)**
- Overlay em `enterprise/` **estende/sobrescreve** o OSS — nunca editar OSS para comportamento Enterprise-only.
- Usar `prepend_mod_with` / `include_mod_with` (módulo no namespace `Enterprise::`).
- Manter **contratos de request/response estáveis** entre OSS e Enterprise.
- Specs Enterprise em `spec/enterprise/`. Buscar arquivos correlatos nas **duas árvores** antes de editar.

**i18n / branding**
- Só editar `en.yml` (backend) e `en.json` (frontend); demais idiomas são da comunidade.
- Strings de marca → preferir `replaceInstallationName` (`shared/composables/useBranding`) em vez de hardcode.

**Regras com impacto direto em roles/permissões**
1. Qualquer mudança em permissões granulares é **código no overlay `enterprise/`** → vai via injeção, não no OSS. (Licenciamento: tratado como **MIT**, igual ao OSS — ver §6.)
2. O CI da Community Edition **remove a pasta `enterprise/`** e roda só a OSS → a OSS **não pode depender** de nada em `enterprise/`.
3. Novas strings de UI/labels de permissão só em `en.json` / `en.yml`.
4. Contratos de API (payload `permissions`, `role`, `custom_role_id`) precisam continuar estáveis OSS↔Enterprise.

---

## 2. Mapa de arquitetura (Fase 1)

**Stack real (confirmado)**

| Camada | Tecnologia | Evidência |
|---|---|---|
| Ruby | **3.4.4** | `.ruby-version`, `Gemfile:3` |
| Backend | **Rails `~> 7.1`** | `Gemfile:7` |
| Node / pkg | Node 24.x, pnpm 10.x | `.nvmrc`, `package.json` |
| Frontend | **Vue 3** (`^3.5.12`) + **Vite 6** | `package.json`, `config/vite.json` |
| State | **Vuex `~4.1`** (principal) + **Pinia** (em migração) | `dashboard/store/index.js`; `dashboard/stores/*.js` |
| Banco | **PostgreSQL** (+ pgvector/`neighbor`) | `Gemfile` (`pg`, `pgvector`, `neighbor`) |
| Fila / jobs | **Sidekiq 7** (+ sidekiq-cron) | `Gemfile` |
| Cache / pub-sub | **Redis** | `Gemfile` |
| WebSockets | **ActionCable** | `config/cable.yml` |
| AuthN | **Devise** + `devise_token_auth` + 2FA | `Gemfile` |
| AuthZ | **Pundit** | `Gemfile`, `app/policies/` |
| Eventos internos | **Wisper** | `app/dispatchers/`, `app/listeners/` |
| Super Admin | **Administrate** | `app/dashboards/` |

**Estrutura de pastas**
- `app/` (OSS, MIT): camadas além do MVC — `builders/`, `finders/`, `services/`, `dispatchers/` + `listeners/` (Wisper), `presenters/`, `policies/`, `views/` (jbuilder).
- `enterprise/` (overlay Enterprise; **MIT**, igual ao OSS — ver §6; a distinção OSS/overlay aqui é técnica, não de licença): **espelha** a árvore `app/` (`enterprise/app/models`, `.../policies`, `.../controllers`, `.../services`, `.../views`...). Confirmado via `ls`.
- `lib/`: `chatwoot_app.rb` (`ChatwootApp.enterprise?` / `extensions`), `current.rb`, `captain/`, `integrations/`, `seeders/`.
- `config/`: `routes.rb` (~725 linhas), `features.yml`, `initializers/01_inject_enterprise_edition_module.rb` (mecanismo de injeção).
- `spec/`: espelha `app/` + **`spec/enterprise/`**.
- `app/javascript/`: múltiplos SPAs — `dashboard/` (agente), `widget/` (cliente), `portal/` (help center público), `survey/` (CSAT), `shared/`, `design-system/`. Dentro de `dashboard/`: `store/` (Vuex) vs `stores/` (Pinia); `components/` (legado) vs `components-next/` (novo); `routes/`, `composables/`, `helper/`, `constants/`.

**Fluxo de uma requisição típica (exemplo: criar conversa)**
`config/routes.rb` (namespace `api/v1/accounts`, `resources :conversations`) → `app/controllers/api/v1/accounts/conversations_controller.rb` (herda de `...accounts/base_controller.rb`) → **builder** `app/builders/conversation_builder.rb` / **finder** `app/finders/conversation_finder.rb` / **service** `app/services/conversations/...` → **model** `app/models/conversation.rb` → **view jbuilder** `app/views/api/v1/accounts/conversations/*.json.jbuilder` → frontend Vuex em `dashboard/store/modules/conversations/`. (Chatwoot usa **jbuilder**, não ActiveModelSerializers, neste fluxo.)

---

## 3. Mapa de features (Fase 2)

Padrão por domínio: **model → controller de API → frontend**.

| Domínio | Model(s) | Controller(s) | Frontend |
|---|---|---|---|
| Conversas/Mensagens | `app/models/conversation.rb`, `message.rb`, `conversation_participant.rb` | `api/v1/accounts/conversations_controller.rb` (+ subpasta `conversations/`) | `routes/dashboard/conversation/`, `store/modules/conversations/`, `components-next/Conversation/` |
| Contatos | `app/models/contact.rb`, `contact_inbox.rb`, `note.rb` | `api/v1/accounts/contacts_controller.rb` | `routes/dashboard/contacts/`, `store/modules/contacts/` |
| Inboxes/Canais | `app/models/inbox.rb` + `app/models/channel/*` (web_widget, email, whatsapp, facebook, etc.) | `api/v1/accounts/inboxes_controller.rb`, `channels/*` | `routes/dashboard/settings/inbox/`, `store/modules/inboxes/` |
| Agentes/Equipes | `user.rb`, `account_user.rb`, `team.rb`, `team_member.rb`, `inbox_member.rb` | `agents_controller.rb`, `teams_controller.rb`, `team_members_controller.rb` | `settings/agents/`, `settings/teams/`, stores `agents.js`, `teams/` |
| Relatórios | `reporting_event.rb`, `reporting_events_rollup.rb` | `reports_controller.rb`, `summary_reports_controller.rb`, `live_reports_controller.rb` | `settings/reports/`, store `reports.js` |
| Automações | `automation_rule.rb` | `automation_rules_controller.rb` (+ `services/automation_rules/`) | `settings/automation/`, store `automations.js` |
| Base de Conhecimento | `portal.rb`, `category.rb`, `article.rb`, `folder.rb` | `portals_controller.rb`, `articles_controller.rb`, `categories_controller.rb` | `routes/dashboard/helpcenter/`; público em `app/javascript/portal/` |
| Campanhas | `campaign.rb` | `campaigns_controller.rb` | `routes/dashboard/campaigns/`, store `campaigns.js` |
| Macros | `macro.rb` | `macros_controller.rb` (`execute`) | `settings/macros/`, store `macros.js` |
| Custom Attributes | `custom_attribute_definition.rb` | `custom_attribute_definitions_controller.rb` | `settings/attributes/`, store `attributes.js` |
| Labels | `label.rb` (gem acts-as-taggable-on) | `labels_controller.rb` | store `labels.js` |
| CSAT | `csat_survey_response.rb` | `csat_survey_responses_controller.rb` | app `app/javascript/survey/` |
| **SLA (Enterprise)** | `enterprise/app/models/sla_policy.rb`, `applied_sla.rb` | rota `custom` enterprise | `settings/sla/` |
| **Custom Roles (Enterprise)** | `enterprise/app/models/custom_role.rb` | `enterprise/.../custom_roles_controller.rb` | `settings/customRoles/`, store `customRole.js` |
| **Captain/AI (Enterprise)** | `enterprise/app/models/captain/*` | namespace `captain` | `components-next/captain/`, `copilot/` |

---

## 4. Dossiê de Roles & Permissões (Fase 3)

### Visão de uma frase

A role no Chatwoot é **por-conta** (vive em `account_users`, não em `users`): um **enum inteiro** `{ agent: 0, administrator: 1 }`. Toda a autorização OSS deriva de `account_user.administrator?`. A Enterprise **não edita o OSS**: ela **prepende** módulos `Enterprise::*` (gatilhada pela existência da pasta `enterprise/`) que reescrevem `AccountUser#permissions` para devolver permissões granulares de uma **custom role**, e que estendem cada policy com `permissão || super`.

---

### 3A — Roles fixas (OSS, sob `app/` e `db/`)

**Banco de dados** — tabela `account_users` (`db/schema.rb:43-60`):
- `role :integer default(0)` → enum `agent`/`administrator`. Sem constraint no banco (validação vem do enum no model).
- Índice único `uniq_user_id_per_account_id` em `(account_id, user_id)` → uma role por usuário por conta.
- **Ganchos Enterprise já presentes no schema OSS:** `custom_role_id :bigint` (+ índice) e `agent_capacity_policy_id :bigint`.
- A tabela nasce de `db/migrate/20230426130150_init_schema.rb`; a coluna `custom_role_id` é adicionada por `db/migrate/20240726220747_add_custom_roles.rb` (`add_reference :account_users, :custom_role, optional: true`).
- `User` **não tem** coluna `role` própria — a role é sempre relativa à conta.

**Models**
- `app/models/account_user.rb:34` → `enum role: { agent: 0, administrator: 1 }` (gera `administrator?`/`agent?`).
- `app/models/account_user.rb:56-58` → **a ponte role→permissões** (ponto que a Enterprise sobrescreve):
  ```ruby
  def permissions
    administrator? ? ['administrator'] : ['agent']
  end
  ```
- `app/models/account_user.rb:84-86` → ganchos de injeção: `prepend_mod_with('AccountUser')`, `include_mod_with('Concerns::AccountUser')`.
- `app/models/concerns/user_attribute_helpers.rb` (incluído em `User`) → `administrator?`/`agent?`/`role` delegam para `current_account_user` (resolvido via `Current.account`).
- `app/models/account.rb:115-121` → scopes `agents` / `administrators` filtrando `account_users.role`.

**Autorização (Pundit)**
- `app/policies/application_policy.rb:1-10` → o contexto Pundit é um **hash** `{ user, account, account_user }`, **não** o `User`. Decisões saem de `@account_user`. Defaults restritivos (`false`).
- Exemplos: `report_policy.rb` (`view? → @account_user.administrator?`), `contact_policy.rb` (`import?/export?/destroy? → administrator?`; resto `true`), `conversation_policy.rb` (`destroy? → administrator?`; `show?` combina admin/agent/bot), `account_policy.rb`, `inbox_policy.rb`, `user_policy.rb` (governa gestão de agentes — tudo exige admin).
- Contexto montado em `app/controllers/application_controller.rb:21-27` (`pundit_user`); `Current.account_user` resolvido em `app/controllers/concerns/ensure_current_account_helper.rb:21-24`.
- Atalhos sem policy: `app/controllers/api/base_controller.rb:20-22` → `check_admin_authorization?` levanta `Pundit::NotAuthorizedError unless Current.account_user.administrator?`.
- Quase toda policy OSS termina com `XxxPolicy.prepend_mod_with('XxxPolicy')` — **o gancho de extensão Enterprise**.

**API / Controllers**
- `app/controllers/api/v1/accounts/agents_controller.rb` → autoriza via `UserPolicy` (só admin). Strong params **permitem `:role`** (linhas 74-84); os valores válidos são restringidos pelo **enum** (valor inválido levanta `ArgumentError`). `update` seta a role no `current_account_user`, não no `user`.
- `app/builders/agent_builder.rb:12` → role default `:agent`.
- `lib/current.rb` → `Current` com `thread_mattr_accessor :account_user`.

**Frontend**
- Payload de auth — `app/views/api/v1/models/_user.json.jbuilder`: `json.role` (l.16), e por conta `json.role` (l.27) + `json.permissions` (l.28, = `['agent']`/`['administrator']`). **Linha 34**: `json.partial! '...account_user' if ChatwootApp.enterprise?` → gancho que injeta `custom_role_id`.
- Store getters — `dashboard/store/modules/auth.js`: `getCurrentRole` (l.62-68) e `getCurrentCustomRoleId` (l.70-76, gancho).
- `dashboard/composables/useAdmin.js:11-12` → `isAdmin = role === 'administrator'`.
- `dashboard/helper/permissionsHelper.js` → `getUserPermissions` (lê `currentAccount.permissions`) e `getUserRole` (l.19-26: retorna `'custom_role'` se `custom_role_id` presente — gancho).
- `dashboard/constants/permissions.js:10` → `ROLES = ['agent', 'administrator']`.
- Gating de rota — `dashboard/helper/routeHelpers.js` (`routeIsAccessibleFor` lê `route.meta.permissions`) + `dashboard/routes/index.js` (guards). Gating de UI — `dashboard/composables/usePolicy.js` (`checkPermissions` / `shouldShow`).

---

### 3B — Custom Roles / RBAC (Enterprise, sob `enterprise/`)

**Model e permissões** — `enterprise/app/models/custom_role.rb` (lido e verificado):
```ruby
class CustomRole < ApplicationRecord
  belongs_to :account
  has_many :account_users, dependent: :nullify

  PERMISSIONS = %w[
    conversation_manage
    conversation_unassigned_manage
    conversation_participating_manage
    contact_manage
    report_manage
    knowledge_base_manage
  ].freeze

  validates :name, presence: true
  validates :permissions, inclusion: { in: PERMISSIONS }
end
```

**Lista canônica de 6 permissões** (fonte de verdade: `enterprise/app/models/custom_role.rb:31-38`; descrições nos comentários l.19-25):

1. `conversation_manage` — gerencia **todas** as conversas.
2. `conversation_unassigned_manage` — gerencia conversas não atribuídas e pode atribuí-las a si.
3. `conversation_participating_manage` — gerencia conversas em que participa (atribuído ou participante).
4. `contact_manage` — gerencia contatos.
5. `report_manage` — gerencia relatórios.
6. `knowledge_base_manage` — gerencia portais da base de conhecimento.

Espelho no frontend: `app/javascript/dashboard/constants/permissions.js` → `AVAILABLE_CUSTOM_ROLE_PERMISSIONS` (lista idêntica de 6).

**Tabela** `custom_roles` (`db/schema.rb:805-813`): `name`, `description`, `account_id`, **`permissions :text array default([])`** (array de text Postgres, **não jsonb**), índice em `account_id`.

#### Ponto de injeção (o coração da integração) — verificado

**Mecanismo** — `config/initializers/01_inject_enterprise_edition_module.rb` (adaptado do GitLab): faz `Module.prepend(InjectEnterpriseEditionModule)`, dando a toda classe os métodos `prepend_mod_with` / `include_mod_with` / `extend_mod_with`. Dado `Foo.prepend_mod_with('Foo')`, ele percorre `ChatwootApp.extensions`, monta `Enterprise::Foo` e, **se a constante existir**, faz `prepend`.

**Gating físico** — `lib/chatwoot_app.rb:14-44`: `enterprise?` = `root.join('enterprise').exist?` (desligável por `ENV['DISABLE_ENTERPRISE']`); `extensions` retorna `['enterprise']` só nesse caso. **Em OSS puro (CI sem `enterprise/`), `extensions == []` e nada é prependado** — por isso a OSS funciona sozinha.

**O override central** — `enterprise/app/models/enterprise/account_user.rb` (lido e verificado):
```ruby
module Enterprise::AccountUser
  def permissions
    custom_role.present? ? (custom_role.permissions + ['custom_role']) : super
  end
end
```
A associação vem de `enterprise/app/models/enterprise/concerns/account_user.rb` (`belongs_to :custom_role, optional: true`).

**Antes → Depois** (classe OSS `AccountUser`):
- **Antes (OSS puro):** `permissions` devolve `['administrator']` ou `['agent']` (`app/models/account_user.rb:56-58`).
- **Depois (Enterprise prependada):** se há custom role, devolve `custom_role.permissions + ['custom_role']`; senão `super` (cai no OSS). **Este é o ponto de transformação de RBAC fixo → granular.**

**Outras classes OSS estendidas via prepend:**
- `Conversations::PermissionFilterService` → `enterprise/app/services/enterprise/conversations/permission_filter_service.rb` aplica hierarquia de escopo de conversa por permissão (`conversation_manage` > `_unassigned_manage` > `_participating_manage`); senão `super`.
- Policies → módulos `Enterprise::XxxPolicy` com padrão `@account_user.custom_role&.permissions&.include?('xxx_manage') || super`: `ConversationPolicy`, `ContactPolicy` (`contact_manage`), `ReportPolicy` (`report_manage`), `PortalPolicy`/`CategoryPolicy`/`ArticlePolicy` (`knowledge_base_manage`), `CsatSurveyResponsePolicy` (`report_manage`).
- `AgentsController` → `enterprise/app/controllers/enterprise/api/v1/accounts/agents_controller.rb`: após `super`, faz `@agent.current_account_user.update!(custom_role_id: params[:custom_role_id])`.

**Importante:** a própria `CustomRolePolicy` (`enterprise/app/policies/custom_role_policy.rb`) **não** é um override prependado — é policy independente e exige `@account_user.administrator?` em **todas** as ações. Ou seja, **só admin gerencia roles**; uma custom role não pode se auto-conceder gestão de roles.

**Feature flag / gating (3 camadas):**
1. Físico: existência de `enterprise/` (`ChatwootApp.enterprise?`).
2. Feature por conta: `config/features.yml:140-143` → `custom_roles`, `enabled: false`, `premium: true`; listada em `enterprise/config/premium_features.yml`.
3. Plano de billing: `enterprise/app/services/enterprise/billing/reconcile_plan_features_service.rb` menciona `custom_roles` **(conteúdo interno não lido — não confirmado além da menção)**.
- Frontend: `dashboard/featureFlags.js:35` (`CUSTOM_ROLES`); rota `customRoles/customRole.routes.js` exige `featureFlag: CUSTOM_ROLES`, `installationTypes: [CLOUD, ENTERPRISE]`, `permissions: ['administrator']`; paywall em `Index.vue` via getter `accounts/isFeatureEnabledonAccount`.

**API / Controllers Enterprise:**
- Rota `config/routes.rb:124` → `resources :custom_roles, only: [:index, :create, :show, :update, :destroy]`.
- `enterprise/app/controllers/api/v1/accounts/custom_roles_controller.rb` → `before_action :check_authorization` (`authorize(CustomRole)` → `CustomRolePolicy`, só admin); CRUD escopado por `Current.account.custom_roles`; params `permit(:name, :description, permissions: [])`.
- Views jbuilder: `enterprise/app/views/api/v1/models/_custom_role.json.jbuilder` e `_account_user.json.jbuilder` (expõe `custom_role_id` + `custom_role` no payload do agente).

**Frontend Enterprise:**
- Store `dashboard/store/modules/customRole.js`; API client `dashboard/api/customRole.js` (`ApiClient('custom_roles', { accountScoped: true })`).
- UI `dashboard/routes/dashboard/settings/customRoles/`: `Index.vue` (listagem + paywall), `component/CustomRoleModal.vue` (form que itera `AVAILABLE_CUSTOM_ROLE_PERMISSIONS` renderizando um checkbox por permissão; selecionar `conversation_manage` adiciona automaticamente as duas sub-permissões de conversa, espelhando a hierarquia do backend).

---

### 3C — Síntese do fluxo e tabela comparativa

**Fluxo completo de uma checagem de permissão (ex.: agente com custom role tentando ver relatórios):**
1. Banco: `account_users.custom_role_id` aponta para um `custom_roles` com `permissions = ['report_manage', ...]`.
2. Payload de auth (jbuilder OSS l.34 + partial Enterprise) inclui `custom_role_id` e `permissions`.
3. Backend: `AccountUser#permissions` — **Enterprise prepend** devolve `custom_role.permissions + ['custom_role']`.
4. Requisição a relatórios → `ReportPolicy#view?` — **Enterprise prepend** avalia `custom_role.permissions.include?('report_manage') || super`.
5. Frontend: `getUserPermissions` lê `permissions`; `usePolicy`/`routeHelpers` habilitam/escondem rota e botão.
   → O ponto onde a lógica Enterprise é **prependada sobre a OSS** é o passo 3 (model) e o passo 4 (policy). Em OSS puro, esses passos caem direto em `administrator?`.

**Tabela comparativa OSS vs Enterprise por camada**

| Camada | OSS (`app/`) | Enterprise (overlay `enterprise/`) | Conexão |
|---|---|---|---|
| **DB** | `account_users.role` (enum) — `db/schema.rb:43-60` | `custom_roles` table + `account_users.custom_role_id` — `db/schema.rb:805-813` | FK opcional `custom_role_id` (gancho no schema OSS) |
| **Model** | `AccountUser#permissions` → `['agent'\|'administrator']` (`account_user.rb:56-58`); `prepend_mod_with` (l.84) | `Enterprise::AccountUser#permissions` → `custom_role.permissions + ['custom_role']` else `super` (`enterprise/app/models/enterprise/account_user.rb`) | `prepend` reescreve `permissions`; concern adiciona `belongs_to :custom_role` |
| **Policy** | `ReportPolicy#view? → administrator?` etc.; `prepend_mod_with` no rodapé | `Enterprise::ReportPolicy` etc. → `perm.include?('xxx_manage') \|\| super` | `prepend` adiciona caminho granular antes do OSS |
| **Controller** | `AgentsController` (CRUD agente, role via enum) | `Enterprise::...AgentsController` (`super` + seta `custom_role_id`); `CustomRolesController` (CRUD de roles) | `prepend` no Agents; controller dedicado para roles |
| **Frontend** | `ROLES = ['agent','administrator']`; `permissions: [role]`; `useAdmin`/`usePolicy` | `AVAILABLE_CUSTOM_ROLE_PERMISSIONS` (6); `customRole.js` store; UI `settings/customRoles/`; `getUserRole → 'custom_role'` | payload `permissions`/`custom_role_id` unifica os dois modelos |

**Onde a modificação provável cai:** se o objetivo for **mexer nas permissões granulares / custom roles**, é **código no overlay `enterprise/`**. Se for mexer no modelo de roles fixas (agent/administrator) ou nos ganchos/contratos, é **OSS (`app/`)**, e exige cuidado para não quebrar o overlay Enterprise. (Licença: ambas as árvores são tratadas como **MIT** — a distinção é técnica, ver §6.)

---

## 5. Pontos de extensão disponíveis (mexer em roles sem quebrar OSS/Enterprise)

1. **`AccountUser#permissions`** (`app/models/account_user.rb:56-58`) — a "ponte" canônica role→permissões; já sobrescrita por `Enterprise::AccountUser`. Estender daqui mantém um único formato de saída (`['...']`).
2. **`prepend_mod_with` / `include_mod_with`** — adicionar novo módulo `Enterprise::*` em `enterprise/app/...` em vez de editar OSS. Já existe gancho em `AccountUser`, nas policies e em `AgentsController`.
3. **`CustomRole::PERMISSIONS`** (`enterprise/app/models/custom_role.rb:31-38`) — lista canônica de permissões; adicionar uma permissão nova começa aqui (+ espelho no frontend `permissions.js` + label i18n em `en.json`).
4. **Policies via prepend** — padrão `perm.include?('xxx_manage') || super` em `enterprise/app/policies/enterprise/*`.
5. **`Conversations::PermissionFilterService`** — ponto de escopo de listagem por permissão (já estendido pela Enterprise).
6. **Frontend**: `constants/permissions.js`, `helper/permissionsHelper.js`, `composables/usePolicy.js` — camada única que consome `permissions`, agnóstica a OSS vs Enterprise.
7. **Feature flag** `custom_roles` (`config/features.yml`) — para gating por conta/plano.

---

## 6. Perguntas em aberto / incertezas

0. **Licenciamento de `enterprise/` — resolvido: tratar como MIT.** Eu havia rotulado `enterprise/` como "proprietário" por conhecimento do modelo padrão do Chatwoot upstream — **sem base em arquivo deste repo**. Verificação: o **único** arquivo de licença é `LICENSE` na raiz, que licencia sob **"MIT Expat"** (`LICENSE:6`); **não existe `enterprise/LICENSE`** e nenhum texto de licença comercial em `enterprise/`. O `LICENSE` começa com *"Portions of this software are licensed as follows:"* mas lista só a MIT — no upstream haveria aqui bullets carve-out para `enterprise/`/`premium/`, ausentes aqui. **Decisão (confirmada por você): todo o código deste repositório, incluindo o overlay `enterprise/`, é tratado como MIT.** A separação OSS (`app/`) vs overlay (`enterprise/`) ao longo deste documento é, portanto, **técnica** (gating por `prepend_mod_with` + feature flag + CI que remove `enterprise/`), **não** de licença.

1. **`reconcile_plan_features_service.rb`** — confirmado apenas que **menciona** `custom_roles`; a lógica exata de ligar/desligar a feature por plano **não foi lida** (não confirmado).
2. **Getter `accounts/isFeatureEnabledonAccount`** (usado no paywall de `Index.vue`) — referenciado, mas a implementação não foi aberta (não confirmado).
3. **`agent_capacity_policy_id`** — segundo gancho Enterprise em `account_users`, relacionado a capacidade de atribuição (SLA/auto-assign), **fora** do escopo de roles; não aprofundado.
4. **Migrations individuais de role** — a coluna `role` nasce do `init_schema` consolidado; não há migration histórica separada (Glob vazio) — esperado em repos que consolidaram o schema.
5. Não abri linha-a-linha **todas** as ~25 policies OSS nem todos os módulos `Enterprise::*Policy`; cobri os representativos do fluxo de roles. Os demais seguem o mesmo padrão (inferência por convenção consistente).

---

## 7. Riscos e pegadinhas ao modificar a área de roles

1. **CI da Community Edition remove `enterprise/`** — a OSS **não pode depender** de classes/módulos Enterprise. Qualquer lógica nova que a OSS precise chamar deve viver em `app/`, com o comportamento granular **apenas** no override Enterprise (via `super`). Teste mentalmente: "isto compila/passa com a pasta `enterprise/` deletada?"
2. **Contratos de API estáveis** — o payload (`role`, `permissions`, `custom_role_id`) é consumido pelo frontend de forma unificada. Mudar o formato de `permissions` quebra tanto OSS quanto Enterprise; manter o array de strings.
3. **Drift OSS ↔ Enterprise** — ao mexer numa policy OSS, verificar o módulo `Enterprise::` espelhado (`prepend`) para não introduzir divergência. Buscar nas duas árvores antes de editar (`rg ... app enterprise`).
4. **Specs** — Enterprise em `spec/enterprise/`; não escrever specs sem pedido explícito (diretriz do projeto).
5. **i18n** — labels de permissão só em `en.json` (frontend) / `en.yml` (backend); demais idiomas são da comunidade.
6. **Feature flag, não bug** — a UI de custom roles pode estar **desligada por plano/flag** (`custom_roles` premium, default `false`), não por defeito. Validar `ChatwootApp.enterprise?` + feature habilitada antes de concluir que "não funciona".
7. **`CustomRolePolicy` só admin** — não afrouxar: custom roles não devem poder gerenciar roles. Manter a checagem `administrator?`.
8. **Enum sem constraint no banco** — valores de `role` são validados só pelo enum no model; um valor inesperado levanta `ArgumentError`. Cuidado ao mexer no mapeamento `{ agent: 0, administrator: 1 }` (índices são persistidos como inteiros).

---

> **Status:** análise concluída, nenhum código alterado. Aguardando você confirmar o escopo da modificação que deseja fazer antes de qualquer implementação.
