# FlowPilot CRM — Documento Técnico-Executivo (Versão Implementável)

## Resumo executivo
Este documento traduz os requisitos obrigatórios em um **plano de implementação real** para um CRM SaaS B2B multi-dispositivo (desktop, tablet e mobile), com foco em:
1. **Adoção rápida** (UX simples e mobile-first).
2. **Governança empresarial** (controle granular por funil e papel).
3. **Escalabilidade operacional** (arquitetura orientada a eventos + tempo real).

---

## 1) Nome e posicionamento

**Produto:** **FlowPilot CRM**  
**Posicionamento:** CRM visual para equipes B2B que precisam de execução comercial diária com controle fino de acesso por pipeline.

**Tese de mercado**
- Ferramentas “enterprise” normalmente sacrificam simplicidade.
- Ferramentas “simples” geralmente não entregam governança por escopo.
- O FlowPilot ocupa esse espaço: **simples para vendedor, controlável para gestão, auditável para TI**.

---

## 2) Mapa de sprints (0 a 5)

> Cadência: 6 sprints de 2 semanas (12 semanas). Critério de pronto por sprint: código em produção de staging + testes automatizados + telemetria mínima.

## Sprint 0 — Arquitetura e fundamentos (2 semanas)
**Objetivo:** reduzir riscos de base.

**Entregáveis**
- Monorepo (frontend + backend + libs compartilhadas).
- CI/CD com deploy automático em `staging`.
- Infra inicial (PostgreSQL, Redis, storage de arquivos, observabilidade).
- Design System v1 (tokens, componentes base, dark/light).
- Contrato da API v1 (`OpenAPI`) + convenções de erro.
- Skeleton de autenticação Google OAuth2.

## Sprint 1 — Auth + Onboarding + RBAC inicial (2 semanas)
**Objetivo:** garantir entrada segura e governança mínima.

**Entregáveis**
- Login Google obrigatório.
- Primeira sessão exige criação de senha fallback.
- Seleção obrigatória de papel padrão (Vendedor/Gerente).
- Sessão JWT + refresh token rotativo.
- Área admin inicial (CRUD de usuários + ativar/desativar).
- RBAC base (`admin`, `gerente`, `vendedor`).

## Sprint 2 — Pipelines + permissões por funil (2 semanas)
**Objetivo:** entregar núcleo do produto.

**Entregáveis**
- CRUD de funis e etapas (criar, renomear, reordenar, excluir).
- Definição de estágio `closed_won` e `closed_lost`.
- Associação usuário-funil.
- Níveis por funil: `read`, `edit_own`, `edit_all`.
- Bloqueios de visibilidade (somente funis associados).

## Sprint 3 — Kanban responsivo + Drawer completo (2 semanas)
**Objetivo:** experiência principal de uso, rápida e fluida.

**Entregáveis**
- Kanban horizontal responsivo (375/768/desktop).
- Drag-and-drop desktop/tablet + fallback mobile por ação “Mover etapa”.
- Atualização em tempo real via WebSocket.
- Drawer com edição, histórico, próximas atividades, anexos e logs.
- FAB no mobile para criação rápida.

## Sprint 4 — Diferenciais de valor (2 semanas)
**Objetivo:** elevar produtividade.

**Entregáveis**
- Google Calendar (fase 1: CRM → Google, fase 2: bi-direcional em feature flag).
- Smart Docs PDF (template padrão).
- Relatórios visuais essenciais.
- Sales Assistant v1 (regras + IA para sugestões).
- Speech-to-Text mobile (beta).

## Sprint 5 — Hardening, LGPD e produção (2 semanas)
**Objetivo:** prontidão de go-live.

**Entregáveis**
- 2FA opcional para admin (TOTP).
- Exportação e anonimização de dados (LGPD).
- Auditoria completa de acessos e mudanças críticas.
- Testes E2E em Chrome e Safari mobile.
- Teste de carga do Kanban (concorrência realista).
- Go-live checklist + runbook operacional.

---

## 3) Arquitetura simplificada (implementável)

```text
[Web/Mobile Web App]
  | REST (HTTPS) + WS
  v
[API Gateway / BFF]
  |--> [Auth Service]
  |--> [Access Service (RBAC + ACL por pipeline)]
  |--> [Pipeline Service]
  |--> [Deal Service]
  |--> [Activity Service]
  |--> [Report Service]
  |--> [Assistant Service]
  |--> [Integration Service (Google Calendar)]
  |
  |--> [PostgreSQL]
  |--> [Redis Cache]
  |--> [Object Storage: anexos e PDFs]
  |--> [Queue/Event Bus: jobs assíncronos]
  |
  --> [Observabilidade: logs, métricas, tracing]
```

### Decisões-chave de engenharia
- **REST para consistência transacional** e simplicidade de integração.
- **WebSocket para colaboração em tempo real** no Kanban.
- **Redis** para cache de leitura quente (pipelines, permissões agregadas).
- **Fila assíncrona** para tarefas custosas (PDF, sync calendar, IA).
- **Multi-tenant lógico por workspace_id** em todas as entidades de negócio.

---

## 4) Modelo de dados detalhado (core)

## 4.1 Entidades principais

### users
- `id (uuid, pk)`
- `workspace_id (uuid, idx)`
- `google_sub (text, nullable, unique por workspace)`
- `email (citext, unique por workspace)`
- `name, avatar_url`
- `password_hash (text, nullable)`
- `is_active (bool default true)`
- `last_login_at, created_at, updated_at`

### roles
- `id, workspace_id, name, is_system_role`

### permissions
- `id, code(unique), description`

### role_permissions
- `role_id, permission_id` (PK composta)

### user_roles
- `user_id, role_id` (PK composta)

### pipelines
- `id, workspace_id, name, created_by, is_active, created_at, updated_at`

### pipeline_stages
- `id, pipeline_id, name, position, stage_type(open|closed_won|closed_lost), is_active`

### pipeline_memberships
- `id, pipeline_id, user_id, access_level(read|edit_own|edit_all), assigned_at`

### organizations
- `id, workspace_id, name, website, owner_user_id`

### contacts
- `id, workspace_id, organization_id, name, email, phone, owner_user_id`

### deals
- `id, workspace_id, pipeline_id, stage_id, organization_id, owner_user_id`
- `title, amount_numeric, currency_code, expected_close_date`
- `status(open|won|lost), lost_reason, created_at, updated_at`

### activities
- `id, workspace_id, deal_id, type(call|meeting|task|email|note)`
- `title, description, due_at, assigned_to, status(pending|done|overdue)`
- `google_event_id, created_at, updated_at`

### deal_stage_history
- `id, deal_id, from_stage_id, to_stage_id, moved_by, moved_at`

### attachments
- `id, deal_id, file_url, file_name, file_size, mime_type, uploaded_by, uploaded_at`

### audit_logs
- `id, workspace_id, actor_user_id, action, resource_type, resource_id, metadata(jsonb), created_at`

## 4.2 Regras de autorização

**Regra 1 — Visibilidade de funil:**
- Um usuário só enxerga funis presentes em `pipeline_memberships`.

**Regra 2 — Ação por nível de acesso:**
- `read`: apenas GET.
- `edit_own`: POST/PATCH/DELETE somente se `deal.owner_user_id == current_user.id`.
- `edit_all`: POST/PATCH/DELETE para qualquer deal no funil associado.

**Regra 3 — Papel admin:**
- `admin` pode gerenciar usuários, papéis, permissões e associações.

**Regra 4 — Auditoria obrigatória:**
- Mudanças de papel/permissão, transferência de owner e movimentações de estágio devem gerar `audit_logs`.

---

## 5) Fluxo de primeira utilização (login Google → primeiro funil)

1. Usuário entra com Google.
2. Backend valida token Google e cria conta (ou atualiza).
3. Se for primeiro login: redireciona para **Criar senha fallback**.
4. Após senha: tela **“Escolha seu papel”** (`vendedor`/`gerente`).
5. Ativa **onboarding guiado** (tour contextual do Kanban).
6. Se papel `gerente`:
   - Botão “Criar primeiro funil”.
   - Cria nome + etapas.
   - Associa usuários com níveis de acesso.
7. Se papel `vendedor`:
   - Entra no funil já associado.
8. Cria primeiro negócio (FAB no mobile / CTA desktop).
9. Evento WS publica atualização para membros do funil.

---

## 6) Critérios de aceite

## 6.1 Kanban responsivo

### Funcionais
- Exibir colunas horizontalmente com rolagem suave em tablet/desktop.
- Drag-and-drop funcional em desktop/tablet.
- No mobile estreito, permitir mover card por menu de etapa.
- Card mínimo obrigatório: título, organização, valor, previsão de fechamento, dono.
- Clique/touch abre drawer com edição e histórico.

### Performance
- p95 interação de mover card: **< 100ms** no cliente.
- TTI da tela Kanban: **< 2,5s** em 4G simulado.
- FPS médio durante drag: **>= 45** em dispositivos-alvo.

### Responsividade
- Testado em **375px**, **768px** e **>=1280px**.
- Desktop mostra de **3 a 5 colunas completas** + scroll horizontal.

### Resiliência
- Reconexão WS automática e idempotente.
- Persistência correta em caso de reconexão.

## 6.2 Controle de acesso granulado

### Funcionais
- Usuário não pode listar nem abrir funis sem associação explícita.
- Usuário `read` não pode alterar dados.
- Usuário `edit_own` não altera deals de terceiros.
- Usuário `edit_all` altera deals do funil associado.

### Segurança
- Endpoints diretos por ID devem retornar **403** quando fora de escopo.
- Toda negação de acesso deve ser logada (auditoria de segurança).

### Testes de aceite
- Matriz completa 3 níveis x 4 ações (`list/create/edit/delete`) por endpoint crítico.
- 100% de cenários de autorização cobertos por integração.

---

## 7) Matriz esforço x risco (diferenciais)

| Feature | Esforço | Risco | Observação prática |
|---|---|---|---|
| Sales Assistant com IA | Alto | Alto | iniciar com regras determinísticas + IA assistiva opcional |
| Speech-to-Text mobile | Médio | Médio | qualidade depende de ruído/dispositivo |
| Smart Docs PDF | Médio | Baixo | baixo risco com template único inicial |
| Google Calendar bi-direcional | Alto | Alto | exige idempotência e reconciliação de conflitos |
| Relatórios visuais | Médio | Médio | risco em modelagem métrica e custo de consulta |

---

## 8) Ferramentas de prototipação e usabilidade

### Prototipação
- **Figma** (design system e protótipos).
- **FigJam** (journeys e mapeamento de processos).

### Testes de usabilidade
- **Maze** (task success rate e tempo por tarefa).
- **Hotjar** (mapas de calor e sessão real).
- **UserTesting** (entrevistas moderadas em fluxos críticos).

### Qualidade técnica de UX
- **Storybook** para componentes e estados.
- **Playwright** para regressão E2E cross-breakpoint.
- **Lighthouse CI** para budgets de performance/acessibilidade.

---

## 9) MVP: o que cortar e o que manter

## Manter no MVP (essencial)
1. Google OAuth + senha fallback.
2. RBAC + ACL por funil (`read/edit_own/edit_all`).
3. Múltiplos funis e etapas customizáveis.
4. Kanban responsivo com fallback mobile.
5. Drawer com atividades, notas, anexos e histórico.
6. Relatórios essenciais (conversão, tempo por etapa, previsão básica).
7. LGPD mínima: exportar, anonimizar, auditar acesso.

## Cortar no MVP (fase 2)
1. IA generativa completa (manter apenas sugestões por regra).
2. Speech-to-Text amplo (lançar beta por feature flag).
3. Calendar bi-direcional total (começar unidirecional CRM → Google).
4. Smart Docs avançado multi-template.
5. Dashboards analíticos avançados.

**Racional:** MVP de CRM ganha no uso diário. Sem Kanban rápido + permissão correta, não há retenção.

---

## 10) Backlog técnico inicial (épicos e APIs)

## Épicos
- E1: Auth e sessões
- E2: Administração e permissões
- E3: Pipelines e Kanban
- E4: Atividades e agenda
- E5: Diferenciais (IA/STT/PDF)
- E6: LGPD e observabilidade

## Endpoints v1 (amostra)
- `POST /api/v1/auth/google/callback`
- `POST /api/v1/auth/set-password`
- `POST /api/v1/auth/2fa/enable`
- `GET /api/v1/users`
- `POST /api/v1/pipelines`
- `PATCH /api/v1/pipelines/{id}`
- `POST /api/v1/pipelines/{id}/memberships`
- `GET /api/v1/deals?pipeline_id={id}`
- `PATCH /api/v1/deals/{id}/stage`
- `POST /api/v1/deals/{id}/activities`
- `POST /api/v1/deals/{id}/attachments`
- `GET /api/v1/reports/conversion`

## Eventos WS (amostra)
- `deal.created`
- `deal.updated`
- `deal.stage_changed`
- `activity.created`

---

## 11) NFRs e SLOs iniciais

- Disponibilidade API: **99,9% mensal**.
- p95 API leitura crítica (`GET /deals`): **< 300ms**.
- p95 API escrita crítica (`PATCH /deals/{id}/stage`): **< 500ms**.
- RPO: 15 min, RTO: 60 min.
- Auditoria: 100% das operações sensíveis registradas.

---

## 12) Recomendação final ao board/CTO

1. **Apostar em excelência operacional do core** (Kanban + permissões + mobile).
2. **Liberar diferenciais por feature flag** para controlar risco e custo.
3. **Instrumentar adoção desde o dia 1** (ativação, retenção e produtividade).
4. **Tratar segurança/LGPD como produto**, não como pós-projeto.

Se executado nessa ordem, o FlowPilot CRM entra no mercado com proposta clara: **produtividade diária com governança enterprise-lite**, maximizando adoção sem perder controle.
