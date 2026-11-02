# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues at https://github.com/wolpatr/lovebird/issues.

`gh` is NOT installed on the dev machine. Use the GitHub REST API with the token from `~/.git-credentials` (line format `https://wolpatr:<token>@github.com`). Never print the token; never commit it.

## Conventions

Header for every call: `Authorization: Bearer <token>`, `Accept: application/vnd.github+json`. Base URL: `https://api.github.com/repos/wolpatr/lovebird`.

- **Create an issue**: `POST /issues` with `{"title": "...", "body": "...", "labels": ["..."]}`
- **Read an issue**: `GET /issues/<n>`
- **List issues**: `GET /issues?state=open`
- **Comment**: `POST /issues/<n>/comments` with `{"body": "..."}`
- **Apply labels**: `POST /issues/<n>/labels` with `{"labels": ["..."]}`
- **Claim (assign)**: `PATCH /issues/<n>` with `{"assignees": ["wolpatr"]}`
- **Close**: `PATCH /issues/<n>` with `{"state": "closed"}`

## Sub-issues

Make a ticket a child of the map: `POST /issues/<map>/sub_issues` with `{"sub_issue_id": <child database id>}`. The database id is the numeric `id` field of the child issue (returned when created), not the `#number`.

## Blocking

GitHub's native issue dependencies. Add an edge: `POST /issues/<child>/dependencies/blocked_by` with `{"issue_id": <blocker database id>}`. A ticket is unblocked when every ticket blocking it is closed. If the dependencies API is unavailable, fall back to a `Blocked by: #<n>` line at the top of the child body.

## Wayfinding operations

Used by the wayfinder skill.

- **Map**: a single issue labelled `wayfinder:map`. Body holds Destination / Notes / Decisions so far / Not yet specified / Out of scope. Open tickets are never listed in the body; they are found by query.
- **Tickets**: child issues of the map, each labelled `wayfinder:<type>` where type is `research`, `prototype`, `grilling` or `task`. Read the label, not the body: the body never states the type.
- **Frontier**: the map's open children with no assignee and no open blockers. First in map order wins when choosing.
- **Claim**: assign the ticket to wolpatr before any work. An open unassigned ticket is unclaimed.
- **Resolve**: post the answer as a resolution comment, close the issue, then append a one-line pointer (gist + link) to the map's "Decisions so far".
- **Research tickets**: resolve on a throwaway `research/<name>` branch. Push the branch, never merge it, open no PR.
- One resolution per session, with the exception of research tickets, which may run as parallel subagents.