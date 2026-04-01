# Automated Timesheet from Activity Logs

Use Claude to reconstruct timesheets from application activity logs, replacing the manual
process of reviewing raw timing data and mapping it to projects and topics.

## Problem

Filling out timesheets each period takes a long time. Most of that time is spent
reconstructing what I worked on by reviewing data from my Timing app, which records which
applications are open and their window titles. I then manually map that activity to events,
topics, and projects in a Grist document.

## Idea

Have Claude automate this mapping by:

1. Ingesting exported activity logs (application names, window titles, timestamps).
2. Matching activity to the correct topic/project based on context — e.g., which codebase
   was open, which file was being edited, what meeting was happening.
3. Writing the results into my Grist timesheet document.

The hard part is giving Claude enough context to assign activity to the right topic or
project. This requires knowing which codebases and files are associated with which projects
during a given time period.

## Data sources

### Activity tracking (what I was doing)

- **Timing app** — currently in use. Need to figure out how to export data (likely CSV or
  JSON). Old Timing data would be valuable as training/testing data for building and
  validating the mapping logic.
- **ActivityWatch** — open-source alternative. Tried it before but it either missed some
  activity types or caused performance issues. With Claude's help, those problems should be
  easier to fix now. Being open source makes it easier to extend and integrate.

### Context sources (why I was doing it)

- **Google Calendar** — meetings, scheduled focus time, and events provide strong signals
  for what project time should be assigned to.
- **Portfolio** — gives a broad view of who is working on what, helping Claude understand
  which projects I'm likely contributing to in a given period.
- **Jira** — stories assigned to me indicate active work; stories linked to PRs I'm
  reviewing indicate code review time.
- **Slack message log** — messages I've sent provide context about which projects and people
  I was engaging with at a given time.
- **Location** — days I go into the office vs. working remotely may correlate with different
  types of work or meetings.

## Key challenges

- **Project/topic mapping** — the system needs to know which codebase or file corresponds to
  which project for a given time period. This context isn't in the activity logs themselves.
- **Training data** — exporting historical Timing data would provide a large labeled dataset
  (past manual mappings) for testing and refining the automation.
- **Grist integration** — need a way for Claude to read and write to the Grist document.
  Grist has a REST API that could work for this.

## Pieces to figure out

- How to export data from Timing (CSV, API, database file?)
- How to give Claude access to Grist (API key + MCP server or direct API calls)
- How to represent the project/topic mapping rules so Claude can apply them
- Whether to switch to ActivityWatch for better extensibility
