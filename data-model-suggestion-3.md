# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Team Retrospective Tool · Created: 2026-05-25

## Philosophy

A retrospective session is fundamentally a sequence of real-time collaborative events: a facilitator starts a phase, participants add cards, participants cast votes, the facilitator groups cards into themes, the team discusses themes in priority order, and action items emerge from discussion. These events happen over WebSocket connections with multiple concurrent users. An event-sourced model captures every interaction as an immutable event, making the event log the single source of truth and the board state a projection that can be rebuilt at any point.

This architecture is particularly natural for retrospectives because: (1) real-time collaboration already requires broadcasting events to connected clients — the event store IS the broadcast log; (2) anonymous feedback requires knowing WHO voted but showing only THAT votes were cast — events store the full picture while read models project the anonymized view; (3) facilitators need session timelines ("what happened and when") for post-session reports; (4) AI features need the full interaction history (vote patterns, discussion time per theme, card edit history) to generate meaningful insights.

Action item carry-forward, health check trends, and cross-sprint analytics all benefit from temporal queries that event sourcing handles naturally — "what was the team's health score trajectory?" is a projection over health check events, not a snapshot query.

**Best for:** Teams building a real-time collaborative retrospective platform where WebSocket event broadcasting, full session replay, facilitator analytics, and AI-powered interaction analysis are core differentiators.

**Trade-offs:**
- **Pro:** Event log doubles as WebSocket broadcast history — replay for late joiners
- **Pro:** Full audit trail of who added/edited/deleted cards, even in anonymous mode
- **Pro:** Session timeline reconstructable from events — facilitator reports write themselves
- **Pro:** AI has rich interaction data (vote timing, discussion duration, edit frequency)
- **Pro:** Health check trends are projections over events — no separate analytics pipeline
- **Con:** Board state requires projection from events — more complex read path
- **Con:** "Delete my card" must be handled carefully (soft-delete event, not event removal)
- **Con:** Event schema versioning needed as the product evolves
- **Con:** 14 tables including read models — moderate complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Scrum Guide 2020 | Session phase events align with Sprint Retrospective ceremony flow |
| RFC 6455 (WebSocket) | Events are the native WebSocket message format — store what you broadcast |
| CloudEvents 1.0 | Event envelope structure (ce_source, ce_type, ce_specversion) |
| OAuth 2.0 (RFC 6749) | Integration auth events for Jira/Slack token lifecycle |
| SCIM 2.0 (RFC 7644) | User provisioning events |
| GDPR | Event immutability with crypto-erasure for right-to-deletion |
| ISO 8601 | All timestamps as TIMESTAMPTZ |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    version BIGINT NOT NULL,
    data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- metadata: {
    --   "actor_id": "uuid",
    --   "actor_ip": "192.168.1.1",
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid",
    --   "websocket_session_id": "uuid",
    --   "ce_source": "retro-tool/sessions",
    --   "ce_specversion": "1.0"
    -- }
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_store_2026_q1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE event_store_2026_q2 PARTITION OF event_store
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_q3 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE event_store_2026_q4 PARTITION OF event_store
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_events_actor ON event_store((metadata->>'actor_id'), occurred_at);
CREATE INDEX idx_events_correlation ON event_store((metadata->>'correlation_id'));
```

---

## Event Type Registry

```sql
CREATE TABLE event_types (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Session lifecycle events
INSERT INTO event_types (event_type, stream_type, description) VALUES
('session.created',           'session', 'Retrospective session created with format and settings'),
('session.scheduled',         'session', 'Session scheduled for a future date'),
('session.started',           'session', 'Facilitator started the session'),
('session.phase_changed',     'session', 'Facilitator advanced to next phase (collecting→voting→discussing→actions→completed)'),
('session.timer_started',     'session', 'Facilitator started a countdown timer for current phase'),
('session.timer_expired',     'session', 'Phase timer expired'),
('session.settings_updated',  'session', 'Session settings modified (vote limit, anonymity, etc.)'),
('session.completed',         'session', 'Session marked as completed by facilitator'),

-- Card events
('card.added',                'session', 'Participant added a card to a column'),
('card.edited',               'session', 'Participant edited their card content'),
('card.deleted',              'session', 'Participant removed their card'),
('card.moved',                'session', 'Card moved to a different column'),
('card.grouped',              'session', 'Card assigned to a theme by facilitator'),
('card.ungrouped',            'session', 'Card removed from a theme'),
('card.reordered',            'session', 'Card sort order changed within column'),

-- Voting events
('vote.cast',                 'session', 'Participant voted on a card'),
('vote.removed',              'session', 'Participant removed their vote from a card'),
('vote.limit_reached',        'session', 'Participant hit their max votes for this session'),

-- Theme events
('theme.created',             'session', 'Facilitator created a new theme/group'),
('theme.renamed',             'session', 'Theme title changed'),
('theme.deleted',             'session', 'Theme removed, cards ungrouped'),
('theme.reordered',           'session', 'Theme discussion order changed'),
('theme.discussion_started',  'session', 'Facilitator opened discussion on this theme'),
('theme.discussion_ended',    'session', 'Discussion on this theme concluded'),

-- Action item events
('action.created',            'session', 'Action item created from discussion'),
('action.assigned',           'session', 'Action item assigned to an owner'),
('action.updated',            'session', 'Action item title/description/priority changed'),
('action.status_changed',     'session', 'Action item status changed (pending→in_progress→completed)'),
('action.carried_forward',    'session', 'Uncompleted action carried to next session'),
('action.synced_external',    'session', 'Action item synced to Jira/GitHub/Linear'),
('action.due_date_set',       'session', 'Due date set or changed on action item'),

-- Health check events
('health_check.started',      'health_check', 'Health check initiated for a team'),
('health_check.response',     'health_check', 'Individual user submitted dimension scores'),
('health_check.completed',    'health_check', 'All responses collected, summary generated'),

-- AI events
('ai.sentiment_analyzed',     'session', 'Sentiment analysis completed for session cards'),
('ai.themes_suggested',       'session', 'AI suggested theme groupings for cards'),
('ai.action_scored',          'session', 'AI scored action item effectiveness based on history'),
('ai.facilitation_tip',       'session', 'AI generated facilitation suggestion'),
('ai.cross_team_pattern',     'session', 'AI detected cross-team pattern'),

-- Team and workspace events
('team.created',              'team', 'Team created in workspace'),
('team.member_added',         'team', 'User added to team'),
('team.member_removed',       'team', 'User removed from team'),
('team.member_role_changed',  'team', 'Team member role changed (member↔facilitator)'),

-- Integration events
('integration.connected',     'workspace', 'External integration connected (Jira, Slack, etc.)'),
('integration.disconnected',  'workspace', 'External integration disconnected'),
('integration.sync_completed','workspace', 'Integration sync cycle completed'),
('integration.sync_failed',   'workspace', 'Integration sync failed');
```

---

## Event Data Examples

```sql
-- session.created
-- data: {
--   "team_id": "uuid",
--   "facilitator_id": "uuid",
--   "title": "Sprint 24 Retro",
--   "sprint_name": "Sprint 24",
--   "format": {
--     "name": "Start / Stop / Continue",
--     "columns": [
--       {"key": "start", "label": "Start", "prompt": "What should we start doing?"},
--       {"key": "stop", "label": "Stop", "prompt": "What should we stop doing?"},
--       {"key": "continue", "label": "Continue", "prompt": "What should we keep doing?"}
--     ]
--   },
--   "settings": {"is_anonymous": true, "max_votes_per_user": 5, "timer_seconds": 300}
-- }

-- card.added
-- data: {
--   "card_id": "uuid",
--   "column_key": "stop",
--   "content": "Deploying on Fridays — we had two incidents this sprint"
-- }

-- vote.cast
-- data: {
--   "card_id": "uuid",
--   "voter_id": "uuid",
--   "votes_remaining": 3
-- }

-- card.grouped
-- data: {
--   "card_id": "uuid",
--   "theme_id": "uuid",
--   "theme_title": "Deployment process"
-- }

-- action.carried_forward
-- data: {
--   "action_id": "uuid",
--   "from_session_id": "uuid",
--   "to_session_id": "uuid",
--   "title": "Set up deployment freeze for Fridays",
--   "times_carried": 2,
--   "original_session_id": "uuid"
-- }

-- health_check.response
-- data: {
--   "health_check_id": "uuid",
--   "scores": {
--     "teamwork": {"score": 4, "trend": "up", "comment": "Pairing sessions helped"},
--     "process": {"score": 3, "trend": "same"},
--     "tech_quality": {"score": 2, "trend": "down", "comment": "Too much tech debt"}
--   }
-- }
```

---

## Read Models

```sql
CREATE TABLE rm_sessions (
    session_id UUID PRIMARY KEY,
    team_id UUID NOT NULL,
    facilitator_id UUID NOT NULL,
    title TEXT,
    sprint_name TEXT,
    status TEXT NOT NULL DEFAULT 'draft',
    format JSONB NOT NULL,
    settings JSONB NOT NULL,
    themes JSONB NOT NULL DEFAULT '[]',
    card_count INT NOT NULL DEFAULT 0,
    vote_count INT NOT NULL DEFAULT 0,
    action_count INT NOT NULL DEFAULT 0,
    participant_count INT NOT NULL DEFAULT 0,
    ai JSONB,
    scheduled_at TIMESTAMPTZ,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    last_event_version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_sessions_team ON rm_sessions(team_id, started_at DESC);

CREATE TABLE rm_boards (
    card_id UUID PRIMARY KEY,
    session_id UUID NOT NULL,
    author_id UUID NOT NULL,
    column_key TEXT NOT NULL,
    content TEXT NOT NULL,
    theme_id UUID,
    theme_title TEXT,
    vote_count INT NOT NULL DEFAULT 0,
    voter_ids UUID[] NOT NULL DEFAULT '{}',
    sort_order INT NOT NULL DEFAULT 0,
    sentiment TEXT,
    sentiment_score REAL,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_boards_session ON rm_boards(session_id, column_key)
    WHERE NOT is_deleted;
CREATE INDEX idx_rm_boards_fulltext ON rm_boards
    USING GIN (to_tsvector('english', content));

CREATE TABLE rm_action_items (
    action_id UUID PRIMARY KEY,
    session_id UUID NOT NULL,
    team_id UUID NOT NULL,
    card_id UUID,
    title TEXT NOT NULL,
    description TEXT,
    owner_id UUID,
    status TEXT NOT NULL DEFAULT 'pending',
    priority TEXT NOT NULL DEFAULT 'medium',
    due_date DATE,
    times_carried INT NOT NULL DEFAULT 0,
    original_session_id UUID,
    external_id TEXT,
    external_source TEXT,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_actions_team ON rm_action_items(team_id, status);
CREATE INDEX idx_rm_actions_owner ON rm_action_items(owner_id)
    WHERE status NOT IN ('completed', 'cancelled');
CREATE INDEX idx_rm_actions_overdue ON rm_action_items(due_date)
    WHERE status = 'pending' AND due_date IS NOT NULL;

CREATE TABLE rm_health_trends (
    team_id UUID NOT NULL,
    health_check_id UUID NOT NULL,
    session_id UUID,
    dimension TEXT NOT NULL,
    avg_score REAL NOT NULL,
    min_score INT NOT NULL,
    max_score INT NOT NULL,
    respondent_count INT NOT NULL,
    trend_vs_previous REAL,
    checked_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (team_id, health_check_id, dimension)
);

CREATE INDEX idx_rm_health_team ON rm_health_trends(team_id, checked_at DESC);

CREATE TABLE rm_session_timelines (
    session_id UUID NOT NULL,
    event_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    phase TEXT NOT NULL,
    summary TEXT NOT NULL,
    actor_id UUID,
    occurred_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (session_id, occurred_at, event_id)
);

CREATE INDEX idx_rm_timeline_session ON rm_session_timelines(session_id, occurred_at);
```

---

## Reference Tables

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

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_event_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE stream_snapshots (
    stream_type TEXT NOT NULL,
    stream_id UUID NOT NULL,
    version BIGINT NOT NULL,
    state JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

---

## Example Queries

### Replay a session's board state from events

```sql
SELECT event_type, data, occurred_at
FROM event_store
WHERE stream_type = 'session'
  AND stream_id = 'session-uuid'
  AND event_type IN ('card.added', 'card.edited', 'card.deleted',
                     'card.grouped', 'vote.cast', 'vote.removed',
                     'theme.created', 'theme.renamed')
ORDER BY version;
```

### Session timeline for facilitator report

```sql
SELECT event_type, phase, summary, occurred_at,
       u.name AS actor_name
FROM rm_session_timelines st
LEFT JOIN users u ON u.id = st.actor_id
WHERE st.session_id = 'session-uuid'
ORDER BY st.occurred_at;
```

### Action item carry-forward chain from events

```sql
SELECT data->>'title' AS title,
       data->>'from_session_id' AS from_session,
       data->>'to_session_id' AS to_session,
       (data->>'times_carried')::INT AS times_carried,
       occurred_at
FROM event_store
WHERE event_type = 'action.carried_forward'
  AND data->>'action_id' = 'action-uuid'
ORDER BY occurred_at;
```

### Health check dimension trends from read model

```sql
SELECT dimension, checked_at::DATE AS date,
       avg_score, respondent_count, trend_vs_previous
FROM rm_health_trends
WHERE team_id = 'team-uuid'
ORDER BY checked_at, dimension;
```

### Vote pattern analysis for AI

```sql
SELECT data->>'voter_id' AS voter_id,
       data->>'card_id' AS card_id,
       occurred_at,
       occurred_at - LAG(occurred_at) OVER (
           PARTITION BY data->>'voter_id' ORDER BY occurred_at
       ) AS time_between_votes
FROM event_store
WHERE stream_type = 'session'
  AND stream_id = 'session-uuid'
  AND event_type = 'vote.cast'
ORDER BY occurred_at;
```

### Average session duration by phase

```sql
SELECT data->>'to_phase' AS phase,
       AVG(
           EXTRACT(EPOCH FROM (
               LEAD(occurred_at) OVER (PARTITION BY stream_id ORDER BY version)
               - occurred_at
           )) / 60
       ) AS avg_minutes
FROM event_store
WHERE event_type = 'session.phase_changed'
  AND stream_id IN (
      SELECT DISTINCT stream_id FROM event_store
      WHERE stream_type = 'session'
        AND data->>'team_id' = 'team-uuid'
        AND event_type = 'session.created'
  )
GROUP BY data->>'to_phase';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 4 | event_store (partitioned), event_types, projection_checkpoints, stream_snapshots |
| Read Models | 5 | rm_sessions, rm_boards, rm_action_items, rm_health_trends, rm_session_timelines |
| Reference | 3 | workspaces, users, teams |
| **Total** | **12** | Plus 4 quarterly partitions on event_store |

---

## Key Design Decisions

1. **Events as WebSocket messages** — Every event in the store corresponds to a WebSocket broadcast message. When a participant adds a card, the `card.added` event is persisted and broadcast simultaneously. Late joiners replay events from the store to catch up. This eliminates the need for a separate real-time messaging layer.

2. **Session stream for all board activity** — Cards, votes, themes, and actions all belong to the `session` stream. This means replaying a single stream reconstructs the entire board state. The alternative — separate streams per card — would require cross-stream projections for the board view.

3. **Anonymous mode via read model projection** — The event store records `actor_id` on every event (required for vote limit enforcement and GDPR deletion). The `rm_boards` read model and API layer strip author identity when `settings.is_anonymous` is true. Anonymity is a projection concern, not a storage concern.

4. **Health check as separate stream** — Health checks are their own aggregate because they can exist independently of sessions (standalone pulse checks) and have a different lifecycle (start → collect responses → generate summary). The `rm_health_trends` read model pre-computes per-dimension averages for trend charts.

5. **Session timeline read model** — `rm_session_timelines` stores a human-readable summary of each significant event in a session. This powers the facilitator report ("14:02 — Collecting phase started. 14:15 — Timer expired. 14:16 — Moved to voting phase.") without replaying raw events.

6. **Vote timing for AI** — Because votes are individual events with timestamps, the AI can analyze vote patterns: how quickly people vote (confidence proxy), whether votes cluster (groupthink detection), which themes get early vs. late votes. This interaction data is unavailable in snapshot-based models.

7. **Action carry-forward as events** — Each `action.carried_forward` event creates a link between sessions. The carry-forward chain is reconstructed by querying events for a given action_id, ordered by time. The read model (`rm_action_items.times_carried`) provides the quick count.

8. **Quarterly partitions** — Retrospective data is less voluminous than monitoring or status page data, but partitioning by quarter keeps event store queries efficient and enables cold storage of older quarters without affecting active session performance.
