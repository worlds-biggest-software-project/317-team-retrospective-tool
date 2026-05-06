# Standards & API Reference

> Project: Team Retrospective Tool · Generated: 2026-05-03

## Industry Standards & Specifications

### Agile & Process Framework Standards

**Scrum Guide 2020 (Schwaber & Sutherland)**
- **URL:** https://scrumguides.org/scrum-guide.html / https://scrumguides.org/docs/scrumguide/v2020/2020-Scrum-Guide-US.pdf
- The authoritative definition of the Sprint Retrospective ceremony. Specifies that a retrospective must inspect individuals, interactions, processes, tools, and the Definition of Done; is timeboxed to a maximum of three hours per one-month Sprint; and produces a plan for the next Sprint. Any tool claiming Scrum compatibility must align with these constraints.

**Kanban Method (Anderson, 2010) — Kanban University**
- **URL:** https://kanban.university/
- Defines service delivery review and operations review practices that are retrospective-adjacent for Kanban teams. Tools serving non-Scrum teams should accommodate these looser, flow-based review formats rather than rigidly enforcing Sprint-based cadences.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework (IETF, 2012)**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749
- The core standard enabling delegated authorization. Required for any integration with Jira, GitHub, Slack, or Microsoft Teams, where the retrospective tool acts as a third-party client obtaining scoped access tokens on behalf of users without storing credentials.

**RFC 7519 — JSON Web Token (JWT) (IETF, 2015)**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7519
- Defines the compact, self-contained token format used for stateless session authentication and inter-service claims. Applicable to the tool's own authentication layer and when consuming JWTs from identity providers.

**RFC 6455 — The WebSocket Protocol (IETF, 2011)**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6455
- Specifies the full-duplex persistent connection protocol used for real-time collaborative retrospective boards (simultaneous card creation, voting, and facilitation updates). All modern browsers support RFC 6455 at 99%+ globally.

**RFC 7644 — SCIM 2.0: System for Cross-domain Identity Management Protocol (IETF, 2015)**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7644
- Defines the REST/JSON protocol for automated user provisioning and de-provisioning. Enterprise deployments require SCIM 2.0 support to synchronise user accounts from identity providers such as Okta, Azure AD, and Google Workspace without manual account management.

**Server-Sent Events (SSE) — WHATWG Living Standard**
- **URL:** https://html.spec.whatwg.org/multipage/server-sent-events.html
- Defines the unidirectional server-push mechanism (text/event-stream media type) useful for broadcasting lightweight real-time updates (e.g., vote counts, card additions) from server to clients, as a lower-overhead alternative to WebSockets for scenarios that do not require bidirectional messaging.

---

### Data Model & API Specifications

**OpenAPI Specification v3.1.0 (OpenAPI Initiative)**
- **URL:** https://spec.openapis.org/oas/v3.1.0.html / https://www.openapis.org/
- The industry standard for describing RESTful APIs in a machine-readable, vendor-neutral format. All public and partner-facing REST endpoints of the retrospective tool should be documented as an OpenAPI 3.1 spec to enable SDK generation, interactive documentation, and third-party integrations.

**JSON Schema (Draft 2020-12) (IETF / OpenJS Foundation)**
- **URL:** https://json-schema.org/
- The specification for validating JSON data structures. Applicable to defining and validating the schemas for retrospective board state, action items, and team health check responses exchanged via the API.

**GraphQL Specification (GraphQL Foundation)**
- **URL:** https://spec.graphql.org/
- Parabol, the primary open-source reference implementation in this space, exposes its entire backend as a GraphQL API. If the tool adopts a similar architecture, the GraphQL spec governs schema design, query/mutation structure, and subscription (real-time) support.

---

### Security & Authentication Standards

**OpenID Connect Core 1.0 (OpenID Foundation)**
- **URL:** https://openid.net/specs/openid-connect-core-1_0.html
- Builds on OAuth 2.0 to add identity layer: standardised ID tokens, UserInfo endpoint, and discovery metadata. Required for SSO (Single Sign-On) with corporate identity providers (Google Workspace, Microsoft Entra ID, Okta).

**GDPR — Regulation (EU) 2016/679, Articles 5 & 25**
- **URL:** https://gdpr-info.eu/ / https://gdpr-info.eu/art-5-gdpr/
- Retrospective data contains personal feedback about colleagues and sentiment data that constitutes personal data under GDPR. Article 5 requires lawfulness, purpose limitation, data minimisation, storage limitation, and confidentiality. Article 25 mandates privacy-by-design: anonymous feedback collection, role-based access controls, configurable data retention policies, and explicit data processor agreements for EU customers.

**OWASP Top 10 (OWASP Foundation)**
- **URL:** https://owasp.org/www-project-top-ten/
- The de-facto checklist for web application security. Directly relevant are A01 (broken access control — preventing team members from viewing other teams' retro data), A02 (cryptographic failures — encrypting sensitive feedback at rest), A07 (identification and authentication failures), and A03 (injection — protecting action item fields and comment inputs).

---

### Accessibility Standards

**Web Content Accessibility Guidelines (WCAG) 2.2 — W3C Recommendation (October 2023)**
- **URL:** https://www.w3.org/TR/WCAG22/
- The international standard for web accessibility. Retrospective tools commonly rely on drag-and-drop, colour coding, and non-labelled interactive elements that fail WCAG 2.2 Level AA success criteria. Compliance requires keyboard-navigable card management, sufficient colour contrast, labelled form controls, and screen-reader accessible board layouts. WCAG 2.2 adds nine new success criteria over 2.1, including focus visibility and accessible authentication.

**ISO 9241-11:2018 — Ergonomics of human-system interaction: Usability definitions and concepts**
- **URL:** https://www.iso.org/standard/63500.html
- Defines usability as effectiveness, efficiency, and satisfaction in a specified context of use. Applicable to evaluating onboarding experience for teams new to structured retrospectives and facilitation flows.

**ISO 9241-210:2019 — Human-centred design for interactive systems**
- **URL:** https://www.iso.org/standard/77520.html
- Provides requirements and recommendations for human-centred design practices. Relevant when designing facilitation UX, anonymity affordances, and progressive disclosure of analytics features.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- **URL:** https://modelcontextprotocol.io/
- An open protocol for connecting AI models to external tools and data sources. An MCP server for the retrospective tool would allow AI coding assistants and AI agents to programmatically trigger retrospectives, query action item status, retrieve trend data, and push findings — enabling AI-native workflows that extend beyond the browser UI.

---

## Similar Products — Developer Documentation & APIs

### TeamRetro

- **Description:** Purpose-built retrospective and team health tool with action item carry-forward, sentiment trend tracking, and integrations with Jira, Slack, and Azure DevOps.
- **API Documentation:** https://groupmap.stoplight.io/docs/teamretro/76440f4735887-team-retro-api
- **SCIM API (Beta):** https://groupmap.stoplight.io/docs/teamretro/YXBpOjI0MjU1MTY3-scim-api
- **Help Portal:** https://help.teamretro.com/article/320-teamretro-api
- **Standards:** REST/JSON; OpenAPI-compatible (hosted on Stoplight)
- **Authentication:** API key (enterprise subscription required); SCIM 2.0 beta for user provisioning
- **Notes:** API is scoped to Read, Write, and SCIM. API keys are managed at account owner level. Rate limits and endpoint inventory are documented on Stoplight.

---

### Parabol (Open Source)

- **Description:** Open-source agile meeting platform covering retrospectives, standups, check-ins, and sprint poker. Used by Netflix and GitHub. AGPL-licensed.
- **GitHub Repository (full source):** https://github.com/ParabolInc/parabol
- **GraphQL Private API README:** https://github.com/ParabolInc/parabol/blob/master/packages/server/graphql/private/README.md
- **Marketplace App:** https://github.com/marketplace/parabol-retrospectives
- **Standards:** GraphQL (Apollo Server); SDL-based schema in .graphql files; REST endpoints for external integrations
- **Authentication:** OAuth 2.0 with third-party providers (GitHub, Google, SAML); JWT-based sessions
- **Notes:** The primary API is GraphQL; admin-only endpoints are available at /admin/graphql (local). Public REST API is limited; most functionality requires reading the open-source codebase. Safe to fork under AGPL.

---

### Atlassian Jira (Sprint Data Source)

- **Description:** The dominant issue-tracking and sprint management platform. Retrospective tools must integrate with Jira to auto-populate sprint metrics (velocity, completed vs. spilled work, cycle time) and to create action items as Jira issues.
- **Jira Cloud Platform REST API v3:** https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/
- **Jira Software Cloud REST API (sprints):** https://developer.atlassian.com/cloud/jira/software/rest/
- **Getting Started:** https://developer.atlassian.com/server/jira/platform/jira-rest-api-tutorials-6291593/
- **Atlassian Forge Platform (for marketplace apps):** https://developer.atlassian.com/platform/forge/
- **Standards:** REST/JSON; OpenAPI 3.0 compliant; Atlassian Document Format (ADF) for rich-text fields in v3
- **Authentication:** OAuth 2.0 (3-legged); API tokens for server-to-server; Forge apps use built-in managed auth
- **Notes:** Jira Software Cloud REST API exposes sprint data at `/rest/agile/1.0/sprint/{sprintId}` and board/backlog endpoints. Forge is the required platform for new Atlassian Marketplace submissions from September 2025 onwards.

---

### GitHub (Sprint & Issue Data Source)

- **Description:** The primary source-code and issue-tracking platform for many engineering teams. Pull request metrics, issue cycle times, and milestone completion are useful contextual data for retrospectives.
- **REST API Documentation:** https://docs.github.com/en/rest
- **Getting Started:** https://docs.github.com/en/rest/using-the-rest-api/getting-started-with-the-rest-api
- **GraphQL API:** https://docs.github.com/en/graphql
- **Octokit SDKs:** https://github.com/octokit (JavaScript, Ruby, .NET)
- **Standards:** REST/JSON (OpenAPI compliant) and GraphQL; GitHub Apps use JWT + installation tokens
- **Authentication:** GitHub Apps (recommended): JWT + OAuth 2.0 installation tokens; Personal Access Tokens (PAT) for simpler integrations; OAuth Apps for user-delegated access
- **Notes:** GitHub Projects (v2) API via GraphQL provides project board data. Issues, PRs, milestones, and review metrics are all queryable and relevant for data-driven retrospective context.

---

### GitLab (Sprint & Issue Data Source)

- **Description:** Alternative DevOps platform serving teams that use GitLab for source control and issue management. Iteration (sprint) and issue data is available via REST API.
- **API Documentation:** https://docs.gitlab.com/api/
- **Developer Portal:** https://developer.gitlab.com/
- **Webhook Documentation:** https://docs.gitlab.com/user/project/integrations/webhooks/
- **Standards:** REST/JSON; webhooks sign payloads with HMAC-SHA256 (GitLab 19.0+); no public GraphQL API for external consumers
- **Authentication:** OAuth 2.0; Personal Access Tokens; Group/Project access tokens
- **Notes:** Iteration endpoints expose sprint-equivalent data at `/groups/:id/iterations` and `/projects/:id/iterations`. Parabol supports GitLab natively, making its integration implementation a useful open-source reference.

---

### Slack (Notifications & Reminders)

- **Description:** The primary team communication platform for engineering teams. Required for retrospective reminders, action item notifications, and meeting scheduling nudges.
- **Developer Documentation:** https://docs.slack.dev/
- **Incoming Webhooks:** https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/
- **Slack Apps (Block Kit, Modals):** https://api.slack.com/block-kit
- **Standards:** REST/JSON (Web API); JSON payload for Block Kit rich messages; incoming webhooks use HTTPS POST
- **Authentication:** OAuth 2.0 (workspace authorization); incoming webhook URL (per-channel); Bot tokens for programmatic posting
- **Notes:** Classic Slack app support is being discontinued in November 2026 — new integrations must use Slack Apps (not legacy webhooks). Block Kit enables rich interactive messages (buttons, select menus) useful for action item follow-up flows.

---

### Microsoft Teams (Notifications & Reminders)

- **Description:** Enterprise communication platform dominant in corporate environments. Required for retrospective notifications and meeting scheduling for Teams-heavy organisations.
- **Developer Documentation:** https://learn.microsoft.com/en-us/microsoftteams/platform/
- **Incoming Webhooks:** https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook
- **Adaptive Cards (rich messages):** https://adaptivecards.io/
- **Standards:** REST/JSON; Adaptive Card schema for rich content; Microsoft Bot Framework for interactive bots
- **Authentication:** Azure Active Directory / Microsoft Entra ID (OAuth 2.0); webhook URLs are channel-scoped
- **Notes:** Workflows (powered by Power Automate) are now the recommended approach for incoming webhooks in 2026, replacing the legacy Office 365 connector model. Adaptive Cards enable interactive action item follow-up within Teams.

---

### Azure DevOps (Sprint Data Source)

- **Description:** Microsoft's DevOps platform. Relevant for organisations using Azure Boards for sprint management alongside Teams for communication.
- **Work REST API (sprints, iterations):** https://learn.microsoft.com/en-us/rest/api/azure/devops/work/?view=azure-devops-rest-7.1
- **Open-Source Retrospectives Extension:** https://github.com/microsoft/vsts-extension-retrospectives
- **Marketplace Extension:** https://marketplace.visualstudio.com/items?itemName=ms-devlabs.team-retrospectives
- **Standards:** REST/JSON; Azure DevOps Services API version 7.1; ODATA for analytics queries
- **Authentication:** Personal Access Tokens (PAT); OAuth 2.0; Azure AD service principals
- **Notes:** Microsoft publishes an open-source retrospectives extension (vsts-extension-retrospectives on GitHub) that demonstrates integration patterns. Sprint capacity, velocity, and iteration data are available via the Work REST API.

---

## Notes

- **No public Echometer API**: Echometer does not publish developer API documentation. Integration with Echometer would require direct partnership or web scraping, neither of which is recommended. Teams needing API access should use TeamRetro or build on Parabol's open-source GraphQL API.

- **EasyRetro API status uncertain**: EasyRetro lists some integrations (Confluence, Slack, Jira export) but has no documented public developer API as of 2026. Not suitable as an integration target.

- **Kollabe API not available publicly**: Kollabe does not publish an API. Its AI template generator is proprietary with no documented endpoints.

- **GDPR and anonymous feedback tension**: Tools storing anonymous retrospective feedback must be careful to ensure that data cannot be de-anonymised (e.g., by timestamp, unique phrasing, or participation metadata). GDPR Article 25 privacy-by-design requirements should inform schema decisions at the data model level.

- **Emerging standard: MCP for AI agents**: The Model Context Protocol is gaining traction as a standard for AI-tool interoperability. Building an MCP server alongside the REST API would allow AI coding assistants to query retrospective insights directly, positioning the tool as AI-native from the outset.
