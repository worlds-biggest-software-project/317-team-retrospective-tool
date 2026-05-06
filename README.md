# Team Retrospective Tool

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source retrospective platform that turns sprint feedback into measurable team improvement.

The Team Retrospective Tool helps agile teams run effective sprint retrospectives, track action items through to completion, and surface trends across sprints and squads. It is built for Scrum Masters, engineering managers, and distributed teams who need a searchable record of feedback and a credible signal of whether committed improvements actually move the needle.

---

## Why Team Retrospective Tool?

- **Incumbents are siloed by ecosystem.** SprintRetro is Jira-only; ClickUp treats retros as templates inside a wider work OS; EasyRetro and Kollabe lack action item tracking and project-management integrations.
- **Most tools stop at the board.** Action items frequently fall through the cracks — only TeamRetro offers reliable carry-forward, and no incumbent correlates past actions with subsequent metric changes.
- **Trend insight is gated behind premium tiers.** Cross-sprint trend tracking is moving to baseline expectation, yet Echometer-style analytics still require consistent paid usage to unlock value.
- **Only one open-source option exists.** Parabol (AGPL) is the sole open-source retrospective tool, and it lacks team health checks, trend analytics, and action effectiveness scoring.
- **AI features are shallow.** Existing AI is limited to template generation (Kollabe), card grouping (Parabol), or report generation (ClickUp) — none combine sentiment trends, action outcome correlation, and cross-team pattern detection.

---

## Key Features

### Retrospective Boards and Formats

- Drag-and-drop board with customizable columns
- Support for standard formats: Start/Stop/Continue, 4Ls, Glad/Sad/Mad, Sailboat, DAKI
- Anonymous voting and feedback collection
- Multiple visualization views (board, list, timeline)
- Export to PDF and CSV for documentation

### Action Item Management

- Action item creation with owner assignment and due dates
- Automatic carry-forward of uncompleted actions to the next retro
- Action item status tracking (completed, in progress, pending)
- Email notifications and reminders for owners
- Jira integration to create action items as native issues

### Trend Analytics and Team Health

- Cross-sprint trend tracking for sentiment, topic frequency, and action completion
- Team health check templates with psychological safety surveys
- Analytics dashboard covering retro frequency, action completion rate, and team health score
- Cross-team analytics and rollup reporting for engineering managers

### Integrations

- Slack and Microsoft Teams for reminders and notifications
- Jira for sprint metrics and issue creation
- GitHub for pulling issues and linking actions
- REST API and webhook support for custom workflows

### Facilitation and Collaboration

- Simple facilitation workflow with clear steps
- Customizable templates and format builder
- Mobile-responsive design
- Free tier for small teams

---

## AI-Native Advantage

The platform uses AI to close gaps that incumbents leave open. AI-generated retro themes draw on commit history, sprint metrics, and previous action items to suggest the most relevant format and starter prompts before each session. Sentiment trend analysis tracks team mood and psychological safety signals over time, alerting managers to burnout or disengagement risk. Action item effectiveness scoring correlates committed actions with subsequent metric changes so teams learn which improvements actually work, while anonymous insight clustering groups semantically similar feedback automatically. Cross-team pattern detection identifies systemic issues — unclear requirements, deployment friction — that recur across squads and escalates them to leadership rather than leaving them isolated.

---

## Tech Stack & Deployment

The project is positioned as an open-source alternative with a self-hosted option, complementing a free tier suitable for small teams. Integrations follow OAuth 2.0 for delegated auth into Jira, GitHub, and Slack, and the platform is designed to comply with GDPR given that retrospective data often contains personal feedback about colleagues. Standard retrospective formats (DAKI, 4Ls, Start/Stop/Continue) are supported as templates, aligning with the Scrum Guide's definition of the Sprint Retrospective ceremony.

---

## Market Context

The broader agile project management tools market exceeds $5 billion globally, with dedicated retrospective tools forming a niche sub-segment that has grown alongside distributed work since 2020. Paid plans for small teams typically run $20–$35/month flat or $5–$8/user/month, with enterprise tiers (analytics, SSO) exceeding $15/user/month. Primary buyers are Scrum Masters and agile coaches, engineering managers at distributed companies, and VPs of Engineering seeking aggregate trend data across multiple squads.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
