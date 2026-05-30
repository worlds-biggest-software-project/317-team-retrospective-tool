# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Team Retrospective Tool · Created: 2026-05-24

## Philosophy

A retrospective tool manages a clear workflow: teams hold retrospective sessions, sessions use board formats with columns, team members add cards to columns, members vote on cards, facilitators group related cards into themes, themes produce action items, and action items carry forward between sprints. Layered on top are team health checks, cross-sprint trend analytics, AI features (sentiment analysis, theme clustering, action effectiveness scoring), and integrations for converting actions into Jira issues. A normalized relational model gives each concept its own table with explicit foreign keys, enabling database-level enforcement of the session→card→vote→action pipeline and clean separation between board content (cards, votes), outcomes (action items), analytics (health checks, trends), and AI enrichment (sentiment, clustering).

This mirrors the conceptual model that Scrum Masters already carry: a retrospective has a format (Start/Stop/Continue, 4Ls), cards are individual feedback items, votes prioritize what matters most, and action items are the commitments that come out of the session. Each maps to a table.

**Best for:** Teams building a production-grade retrospective platform where data integrity across the card→vote→action pipeline is critical, where action item carry-forward needs relational tracking, and where cross-sprint trend analytics are a first-class feature.

**Trade-offs:**
- **Pro:** Database-enforced relationships from session to card to vote to action
- **Pro:** Action items as explicit rows enable carry-forward and completion tracking
- **Pro:** Votes as explicit rows enable voting analytics and anonymity controls
- **Pro:** Health check responses as typed rows enable trend charts
- **Con:** 18 tables — moderate complexity
- **Con:** High join count for "show me the full board with cards, votes, and themes"
- **Con:** Board format configuration varies but is modeled with fixed columns
- **Con:** AI outputs evolve faster than typed columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Scrum Guide 2020 | Sprint Retrospective ceremony structure |
| RFC 6455 (WebSocket) | Real-time board collaboration |
| OAuth 2.0 (RFC 6749) | Jira, GitHub, Slack integration auth |
| SCIM 2.0 (RFC 7644) | Enterprise user provisioning |
| OpenAPI 3.1 | REST API specification |
| GDPR | Anonymous feedback; personal data in health checks |
| WCAG 2.2 | Accessible board interactions |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Workspaces, Teams & Users

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    billing_plan TEXT NOT NULL DEFAULT 'free',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('admin', 'facilitator', 'member')),
    avatar_url TEXT,
    timezone TEXT NOT NULL DEFAULT 'UTC',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_workspace ON users(workspace_id);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('facilitator', 'member')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);
```

---

## Retrospective Sessions

```sql
CREATE TABLE retro_formats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT,
    is_builtin BOOLEAN NOT NULL DEFAULT FALSE,
    columns TEXT[] NOT NULL,
    column_prompts TEXT[],
    icon TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE retro_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    format_id UUID NOT NULL REFERENCES retro_formats(id),
    facilitator_id UUID NOT NULL REFERENCES users(id),
    title TEXT,
    sprint_name TEXT,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'collecting', 'voting', 'discussing', 'actions', 'completed'
    )),
    is_anonymous BOOLEAN NOT NULL DEFAULT TRUE,
    max_votes_per_user INT NOT NULL DEFAULT 5,
    timer_seconds INT,
    scheduled_at TIMESTAMPTZ,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_team ON retro_sessions(team_id, created_at DESC);
CREATE INDEX idx_sessions_status ON retro_sessions(team_id, status)
    WHERE status != 'completed';
```

---

## Cards & Voting

```sql
CREATE TABLE retro_cards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES retro_sessions(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id),
    column_name TEXT NOT NULL,
    content TEXT NOT NULL,
    theme_id UUID REFERENCES card_themes(id),
    vote_count INT NOT NULL DEFAULT 0,
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cards_session ON retro_cards(session_id, column_name);
CREATE INDEX idx_cards_theme ON retro_cards(theme_id) WHERE theme_id IS NOT NULL;
CREATE INDEX idx_cards_fulltext ON retro_cards
    USING GIN (to_tsvector('english', content));

CREATE TABLE card_votes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    card_id UUID NOT NULL REFERENCES retro_cards(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (card_id, user_id)
);

CREATE INDEX idx_votes_card ON card_votes(card_id);

CREATE TABLE card_themes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES retro_sessions(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    color TEXT,
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_themes_session ON card_themes(session_id);
```

---

## Action Items

```sql
CREATE TABLE action_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES retro_sessions(id) ON DELETE CASCADE,
    team_id UUID NOT NULL REFERENCES teams(id),
    card_id UUID REFERENCES retro_cards(id),
    title TEXT NOT NULL,
    description TEXT,
    owner_id UUID REFERENCES users(id),
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'in_progress', 'completed', 'cancelled', 'carried_forward'
    )),
    priority TEXT NOT NULL DEFAULT 'medium' CHECK (priority IN (
        'high', 'medium', 'low'
    )),
    due_date DATE,
    carried_from_id UUID REFERENCES action_items(id),
    carried_to_session_id UUID REFERENCES retro_sessions(id),
    external_id TEXT,
    external_source TEXT,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_actions_session ON action_items(session_id);
CREATE INDEX idx_actions_team ON action_items(team_id, status);
CREATE INDEX idx_actions_owner ON action_items(owner_id) WHERE status NOT IN ('completed', 'cancelled');
CREATE INDEX idx_actions_carried ON action_items(carried_from_id) WHERE carried_from_id IS NOT NULL;
```

---

## Team Health Checks

```sql
CREATE TABLE health_check_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    description TEXT,
    is_builtin BOOLEAN NOT NULL DEFAULT FALSE,
    dimensions TEXT[] NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE health_checks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES retro_sessions(id) ON DELETE CASCADE,
    team_id UUID NOT NULL REFERENCES teams(id),
    template_id UUID NOT NULL REFERENCES health_check_templates(id),
    is_anonymous BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_health_checks ON health_checks(team_id, created_at DESC);

CREATE TABLE health_check_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    health_check_id UUID NOT NULL REFERENCES health_checks(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    dimension TEXT NOT NULL,
    score INT NOT NULL CHECK (score BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (health_check_id, user_id, dimension)
);

CREATE INDEX idx_hc_responses ON health_check_responses(health_check_id);
```

---

## AI & Analytics

```sql
CREATE TABLE ai_analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES retro_sessions(id) ON DELETE CASCADE,
    analysis_type TEXT NOT NULL CHECK (analysis_type IN (
        'sentiment_summary', 'theme_clustering', 'action_effectiveness',
        'cross_team_pattern', 'facilitation_suggestion'
    )),
    content TEXT NOT NULL,
    score REAL,
    details TEXT,
    model_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_session ON ai_analyses(session_id);
```

---

## Integrations & API

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    provider TEXT NOT NULL CHECK (provider IN (
        'jira', 'github', 'slack', 'teams', 'linear', 'asana'
    )),
    status TEXT NOT NULL DEFAULT 'connected',
    credentials_enc TEXT,
    config TEXT,
    last_synced_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, provider)
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Full board with cards, votes, and themes

```sql
SELECT rc.column_name, rc.content, rc.vote_count,
       ct.title AS theme, u.name AS author
FROM retro_cards rc
LEFT JOIN card_themes ct ON ct.id = rc.theme_id
JOIN users u ON u.id = rc.author_id
WHERE rc.session_id = 'session-uuid'
ORDER BY rc.column_name, rc.vote_count DESC;
```

### Action item carry-forward chain

```sql
WITH RECURSIVE carry_chain AS (
    SELECT id, title, status, session_id, carried_from_id, 0 AS depth
    FROM action_items WHERE id = 'action-uuid'
    UNION ALL
    SELECT ai.id, ai.title, ai.status, ai.session_id, ai.carried_from_id, cc.depth + 1
    FROM action_items ai
    JOIN carry_chain cc ON cc.carried_from_id = ai.id
)
SELECT * FROM carry_chain ORDER BY depth DESC;
```

### Team health trend over time

```sql
SELECT hc.created_at::DATE AS date, hcr.dimension,
       AVG(hcr.score) AS avg_score, COUNT(DISTINCT hcr.user_id) AS respondents
FROM health_checks hc
JOIN health_check_responses hcr ON hcr.health_check_id = hc.id
WHERE hc.team_id = 'team-uuid'
GROUP BY hc.created_at::DATE, hcr.dimension
ORDER BY date, dimension;
```

### Action completion rate by team

```sql
SELECT t.name AS team,
       COUNT(*) AS total_actions,
       COUNT(*) FILTER (WHERE ai.status = 'completed') AS completed,
       COUNT(*) FILTER (WHERE ai.status = 'completed') * 100.0 / COUNT(*) AS completion_pct
FROM action_items ai
JOIN teams t ON t.id = ai.team_id
WHERE t.workspace_id = 'workspace-uuid'
GROUP BY t.id, t.name
ORDER BY completion_pct DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Workspace & Users | 4 | workspaces, users, teams, team_members |
| Sessions | 2 | retro_formats, retro_sessions |
| Cards & Voting | 3 | retro_cards, card_votes, card_themes |
| Actions | 1 | action_items (with carry-forward self-reference) |
| Health Checks | 3 | health_check_templates, health_checks, health_check_responses |
| AI | 1 | ai_analyses |
| Integrations | 2 | integrations, api_keys |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Cards with column_name reference** — `retro_cards.column_name` references the column by name from the format's columns array. This avoids a separate `columns` table while preserving the board structure.

2. **Explicit votes** — `card_votes` stores each vote as a row with user and card. This enables vote analytics (who voted, total votes per session) and enforces the `max_votes_per_user` constraint via application logic with database-level uniqueness per card-user pair.

3. **Card themes for grouping** — `card_themes` groups related cards during the discussion phase. The facilitator creates themes and assigns cards to them. This enables theme-based discussion ordering and AI-assisted clustering.

4. **Action item carry-forward** — `action_items.carried_from_id` links carried-forward actions to their origin. `carried_to_session_id` links to the session where the action was carried forward. A recursive CTE traces the full carry-forward chain.

5. **Health check responses per dimension** — `health_check_responses` stores one score per user per dimension (e.g., "Teamwork: 4", "Process: 3"). This enables per-dimension trend charts across sprints.

6. **Retro formats as templates** — `retro_formats` stores both built-in formats (Start/Stop/Continue, 4Ls, Glad/Sad/Mad, Sailboat, DAKI) and custom formats. Columns are stored as a TEXT array.

7. **Session phases as status** — `retro_sessions.status` tracks the facilitation phase (collecting → voting → discussing → actions → completed). The facilitator advances phases manually.

8. **AI analyses per session** — `ai_analyses` stores sentiment summaries, theme clustering results, and action effectiveness scores. Each analysis references the model version for tracking AI accuracy.
