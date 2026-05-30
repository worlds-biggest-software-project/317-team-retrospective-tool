# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Team Retrospective Tool · Created: 2026-05-24

## Philosophy

A retrospective tool's core workflow is simple — sessions contain cards, cards get votes, votes surface priorities, priorities produce action items — but the surrounding configuration is highly variable. Board formats range from 3-column Start/Stop/Continue to 6-zone Sailboat with wind/anchor/rocks/island metaphors. Health check templates vary from Spotify's 10-dimension model to simple 3-question pulse checks. AI outputs evolve rapidly as models improve. A hybrid approach keeps the session→card→action pipeline relational for integrity, while pushing variable configuration, inline collections, and AI outputs into JSONB columns.

The key insight is that a retrospective card's votes, theme assignment, and AI sentiment are all properties of the card — not independent entities that need their own lifecycle. Similarly, a health check's responses are a bounded collection per user that never needs to be queried independently of its parent check. By inlining these as JSONB, the "full board" query becomes a single table scan with no joins, which is exactly how the real-time WebSocket layer will serve data.

**Best for:** Teams building a real-time retrospective tool where board rendering speed matters most, where format flexibility is a product differentiator, and where the development team values rapid iteration over strict normalization.

**Trade-offs:**
- **Pro:** Full board loads in a single query — no joins for cards, votes, themes
- **Pro:** New board formats require zero schema changes — just new JSONB shapes
- **Pro:** AI outputs evolve independently of the relational schema
- **Pro:** Health check templates are infinitely flexible via JSONB dimensions
- **Pro:** 8 tables total — low migration overhead
- **Con:** Vote uniqueness enforcement moves to application layer
- **Con:** Cannot run SQL aggregations directly on inline vote arrays without JSONB unpacking
- **Con:** Action item carry-forward queries require JSONB traversal for history
- **Con:** Card search across sessions requires GIN indexes on JSONB content fields

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Scrum Guide 2020 | Sprint Retrospective ceremony phases modeled as session status |
| RFC 6455 (WebSocket) | JSONB card documents broadcast as-is to WebSocket clients |
| OAuth 2.0 (RFC 6749) | Integration credentials in JSONB config per provider |
| SCIM 2.0 (RFC 7644) | Enterprise user provisioning |
| GDPR | Anonymous mode hides author_id from card JSONB in API responses |
| WCAG 2.2 | Accessible board interactions |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Workspaces, Teams & Users

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings: {
    --   "billing_plan": "pro",
    --   "max_teams": 50,
    --   "features": ["ai_sentiment", "health_checks", "jira_sync"],
    --   "branding": {"logo_url": "...", "primary_color": "#4A90D9"},
    --   "integrations": {
    --     "jira": {"base_url": "...", "credentials_enc": "...", "project_key": "ENG", "last_synced_at": "..."},
    --     "slack": {"webhook_url": "...", "channel": "#retros", "notify_on": ["session_completed", "action_overdue"]},
    --     "github": {"org": "acme", "credentials_enc": "..."}
    --   },
    --   "scim": {"enabled": true, "bearer_token_hash": "..."}
    -- }
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
    members JSONB NOT NULL DEFAULT '[]',
    -- members: [
    --   {"user_id": "uuid", "role": "facilitator", "joined_at": "2026-01-15T10:00:00Z"},
    --   {"user_id": "uuid", "role": "member", "joined_at": "2026-02-01T09:00:00Z"}
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, slug)
);

CREATE INDEX idx_teams_workspace ON teams(workspace_id);
CREATE INDEX idx_teams_members ON teams USING GIN (members jsonb_path_ops);
```

---

## Retrospective Sessions

```sql
CREATE TABLE retro_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    facilitator_id UUID NOT NULL REFERENCES users(id),
    title TEXT,
    sprint_name TEXT,
    status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'collecting', 'voting', 'discussing', 'actions', 'completed'
    )),
    format JSONB NOT NULL,
    -- format: {
    --   "name": "Start / Stop / Continue",
    --   "columns": [
    --     {"key": "start", "label": "Start", "prompt": "What should we start doing?", "color": "#4CAF50"},
    --     {"key": "stop", "label": "Stop", "prompt": "What should we stop doing?", "color": "#F44336"},
    --     {"key": "continue", "label": "Continue", "prompt": "What should we keep doing?", "color": "#2196F3"}
    --   ],
    --   "icon": "arrows"
    -- }
    -- Sailboat example:
    -- format: {
    --   "name": "Sailboat",
    --   "columns": [
    --     {"key": "wind", "label": "Wind", "prompt": "What's pushing us forward?", "color": "#4CAF50", "zone": "positive"},
    --     {"key": "anchor", "label": "Anchor", "prompt": "What's holding us back?", "color": "#F44336", "zone": "negative"},
    --     {"key": "rocks", "label": "Rocks", "prompt": "What risks do we see ahead?", "color": "#FF9800", "zone": "risk"},
    --     {"key": "island", "label": "Island", "prompt": "What's our goal?", "color": "#9C27B0", "zone": "goal"}
    --   ],
    --   "icon": "sailboat",
    --   "layout": "quadrant"
    -- }
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings: {
    --   "is_anonymous": true,
    --   "max_votes_per_user": 5,
    --   "timer_seconds": 300,
    --   "allow_multiple_votes_per_card": false,
    --   "show_author_after_reveal": false,
    --   "card_character_limit": 500
    -- }
    themes JSONB NOT NULL DEFAULT '[]',
    -- themes: [
    --   {"id": "uuid", "title": "Communication gaps", "color": "#FF9800", "sort_order": 0},
    --   {"id": "uuid", "title": "CI/CD improvements", "color": "#4CAF50", "sort_order": 1}
    -- ]
    ai JSONB,
    -- ai: {
    --   "sentiment_summary": {"positive": 0.45, "neutral": 0.35, "negative": 0.20, "generated_at": "..."},
    --   "suggested_themes": [
    --     {"title": "Testing gaps", "card_ids": ["uuid1", "uuid2"], "confidence": 0.87}
    --   ],
    --   "facilitation_tips": ["Several cards mention deploy frequency — consider a focused action item"],
    --   "action_effectiveness": {
    --     "completed_rate": 0.6, "avg_days_to_complete": 8.5,
    --     "comparison": "Below team average of 0.72"
    --   },
    --   "model_version": "gpt-4o-2026-05"
    -- }
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

## Cards

```sql
CREATE TABLE retro_cards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES retro_sessions(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id),
    column_key TEXT NOT NULL,
    content TEXT NOT NULL,
    theme_id TEXT,
    sort_order INT NOT NULL DEFAULT 0,
    votes JSONB NOT NULL DEFAULT '[]',
    -- votes: [
    --   {"user_id": "uuid", "voted_at": "2026-05-24T14:32:00Z"},
    --   {"user_id": "uuid", "voted_at": "2026-05-24T14:33:15Z"}
    -- ]
    ai JSONB,
    -- ai: {
    --   "sentiment": "negative",
    --   "sentiment_score": -0.72,
    --   "suggested_theme": "Communication gaps",
    --   "similar_card_ids": ["uuid1", "uuid2"],
    --   "keywords": ["deploy", "rollback", "downtime"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cards_session ON retro_cards(session_id, column_key);
CREATE INDEX idx_cards_votes ON retro_cards USING GIN (votes jsonb_path_ops);
CREATE INDEX idx_cards_fulltext ON retro_cards
    USING GIN (to_tsvector('english', content));
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
    priority TEXT NOT NULL DEFAULT 'medium' CHECK (priority IN ('high', 'medium', 'low')),
    due_date DATE,
    carry_forward JSONB,
    -- carry_forward: {
    --   "chain": [
    --     {"session_id": "uuid", "sprint_name": "Sprint 22", "status": "carried_forward", "created_at": "2026-04-10"},
    --     {"session_id": "uuid", "sprint_name": "Sprint 23", "status": "carried_forward", "created_at": "2026-04-24"}
    --   ],
    --   "original_session_id": "uuid",
    --   "times_carried": 2
    -- }
    external JSONB,
    -- external: {
    --   "jira": {"issue_key": "ENG-1234", "url": "https://acme.atlassian.net/browse/ENG-1234", "synced_at": "..."},
    --   "github": {"issue_number": 567, "repo": "acme/backend", "url": "...", "synced_at": "..."}
    -- }
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_actions_session ON action_items(session_id);
CREATE INDEX idx_actions_team ON action_items(team_id, status);
CREATE INDEX idx_actions_owner ON action_items(owner_id)
    WHERE status NOT IN ('completed', 'cancelled');
```

---

## Health Checks

```sql
CREATE TABLE health_checks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES retro_sessions(id) ON DELETE CASCADE,
    team_id UUID NOT NULL REFERENCES teams(id),
    template JSONB NOT NULL,
    -- template: {
    --   "name": "Spotify Health Check",
    --   "dimensions": [
    --     {"key": "teamwork", "label": "Teamwork", "description": "We work well together"},
    --     {"key": "process", "label": "Process", "description": "Our process works for us"},
    --     {"key": "tech_quality", "label": "Tech Quality", "description": "We're proud of our code"},
    --     {"key": "value", "label": "Value", "description": "We deliver value to customers"},
    --     {"key": "speed", "label": "Speed", "description": "We ship fast"},
    --     {"key": "fun", "label": "Fun", "description": "We enjoy our work"},
    --     {"key": "learning", "label": "Learning", "description": "We're learning and growing"},
    --     {"key": "mission", "label": "Mission", "description": "We know our mission and it inspires us"},
    --     {"key": "pawns_or_players", "label": "Pawns or Players", "description": "We feel in control"},
    --     {"key": "support", "label": "Support", "description": "We get the support we need"}
    --   ]
    -- }
    is_anonymous BOOLEAN NOT NULL DEFAULT TRUE,
    responses JSONB NOT NULL DEFAULT '[]',
    -- responses: [
    --   {
    --     "user_id": "uuid",
    --     "submitted_at": "2026-05-24T15:00:00Z",
    --     "scores": {
    --       "teamwork": {"score": 4, "trend": "up", "comment": "Better since daily standups improved"},
    --       "process": {"score": 3, "trend": "same"},
    --       "tech_quality": {"score": 2, "trend": "down", "comment": "Tech debt growing"},
    --       "value": {"score": 4, "trend": "up"},
    --       "speed": {"score": 3, "trend": "same"},
    --       "fun": {"score": 5, "trend": "up"},
    --       "learning": {"score": 4, "trend": "same"},
    --       "mission": {"score": 4, "trend": "same"},
    --       "pawns_or_players": {"score": 3, "trend": "down"},
    --       "support": {"score": 4, "trend": "up"}
    --     }
    --   }
    -- ]
    summary JSONB,
    -- summary: {
    --   "averages": {"teamwork": 3.8, "process": 3.2, "tech_quality": 2.5, ...},
    --   "respondent_count": 8,
    --   "lowest_dimension": "tech_quality",
    --   "highest_dimension": "fun",
    --   "trend_vs_last": {"teamwork": +0.3, "process": -0.1, ...}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_health_checks ON health_checks(team_id, created_at DESC);
```

---

## Example Queries

### Full board with cards, votes, and themes

```sql
SELECT rs.format, rs.themes, rs.settings, rs.ai AS session_ai,
       json_agg(
           json_build_object(
               'id', rc.id,
               'column_key', rc.column_key,
               'content', CASE WHEN (rs.settings->>'is_anonymous')::BOOLEAN
                          THEN rc.content ELSE rc.content END,
               'author_id', CASE WHEN (rs.settings->>'is_anonymous')::BOOLEAN
                            THEN NULL ELSE rc.author_id END,
               'vote_count', jsonb_array_length(rc.votes),
               'theme_id', rc.theme_id,
               'ai', rc.ai,
               'sort_order', rc.sort_order
           ) ORDER BY rc.column_key, jsonb_array_length(rc.votes) DESC
       ) AS cards
FROM retro_sessions rs
LEFT JOIN retro_cards rc ON rc.session_id = rs.id
WHERE rs.id = 'session-uuid'
GROUP BY rs.id;
```

### Action item carry-forward history

```sql
SELECT ai.title, ai.status, ai.priority,
       ai.carry_forward->>'times_carried' AS times_carried,
       ai.carry_forward->'chain' AS carry_chain,
       u.name AS owner,
       ai.external->'jira'->>'issue_key' AS jira_key
FROM action_items ai
LEFT JOIN users u ON u.id = ai.owner_id
WHERE ai.team_id = 'team-uuid'
  AND ai.status NOT IN ('completed', 'cancelled')
ORDER BY (ai.carry_forward->>'times_carried')::INT DESC NULLS LAST;
```

### Team health trend over time

```sql
SELECT hc.created_at::DATE AS date,
       dim.key AS dimension,
       AVG((resp->'scores'->dim.key->>'score')::INT) AS avg_score,
       COUNT(DISTINCT resp->>'user_id') AS respondents
FROM health_checks hc,
     jsonb_array_elements(hc.responses) AS resp,
     jsonb_each(hc.template->'dimensions') AS dim(idx, val),
     LATERAL (SELECT val->>'key' AS key) AS dim(key)
WHERE hc.team_id = 'team-uuid'
GROUP BY hc.created_at::DATE, dim.key
ORDER BY date, dimension;
```

### Cards with negative sentiment across sessions

```sql
SELECT rs.sprint_name, rc.content, rc.ai->>'sentiment_score' AS sentiment
FROM retro_cards rc
JOIN retro_sessions rs ON rs.id = rc.session_id
WHERE rs.team_id = 'team-uuid'
  AND (rc.ai->>'sentiment_score')::REAL < -0.5
ORDER BY rs.created_at DESC, (rc.ai->>'sentiment_score')::REAL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Workspace & Users | 3 | workspaces (with integrations JSONB), users, teams (with members JSONB) |
| Sessions | 1 | retro_sessions (with format, themes, AI, settings as JSONB) |
| Cards | 1 | retro_cards (with votes and AI as JSONB) |
| Actions | 1 | action_items (with carry-forward chain and external links as JSONB) |
| Health Checks | 1 | health_checks (with template, responses, summary as JSONB) |
| Integrations | 1 | Inline on workspaces.settings JSONB |
| **Total** | **8** | Including inline integration config (vs. 16 in normalized model) |

---

## Key Design Decisions

1. **Board format as JSONB** — `retro_sessions.format` embeds the full format definition (columns, prompts, colors, layout) directly on the session. This means the same session preserves its format even if the template is later edited, and exotic formats (Sailboat quadrant layout, DAKI with 4+ zones) need no schema changes.

2. **Votes inline on cards** — `retro_cards.votes` stores each vote as a JSONB array entry. Vote count is `jsonb_array_length(votes)`. The trade-off: uniqueness enforcement (one vote per user per card) moves to the application layer, but the board renders without a JOIN.

3. **Themes inline on session** — `retro_sessions.themes` stores theme definitions as a JSONB array. Cards reference themes by ID string (`retro_cards.theme_id`). This avoids a separate table while keeping theme→card grouping intact.

4. **Action carry-forward as JSONB chain** — Instead of a self-referential foreign key, `action_items.carry_forward` contains the full chain history as a JSONB array. This trades recursive CTE queries for simple JSONB reads, at the cost of denormalization when an action is carried forward (the chain array must be appended).

5. **Health check responses inline** — `health_checks.responses` stores all user responses as a JSONB array. This is appropriate because responses are always read and written as a collection — you never query a single user's response independently of the check.

6. **Team members as JSONB** — `teams.members` stores the membership list inline. Teams are small (typically 5-12 members), membership changes infrequently, and the member list is always loaded with the team. The GIN index enables "find all teams for user X" queries.

7. **Integration config on workspace** — Instead of a separate integrations table, provider configurations live in `workspaces.settings.integrations`. Each provider (Jira, Slack, GitHub) has its own JSONB sub-object. This works because integrations are workspace-scoped and always loaded with workspace settings.

8. **AI outputs on cards and sessions** — AI sentiment, theme suggestions, and facilitation tips are stored as JSONB on the entities they describe. This allows AI output schemas to evolve without migrations and keeps AI data co-located with the content it analyzes.
