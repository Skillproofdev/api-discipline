# Product brief: Trackle — team task tracker (v1 backend)

Trackle is a small-team task tracker (think a lightweight Asana). We are
building the first public backend API that our own web client and future
third-party integrations will use. You are designing the HTTP API.

## Domain

- **Projects.** A workspace contains many projects. A project has a name, an
  optional description, a color label, an archived flag, and timestamps. Users
  create, rename, archive, and delete projects.
- **Members.** People belong to a project with a role (`owner`, `editor`,
  `viewer`). Owners invite members by email, change roles, and remove members.
  A member list can get long in big client accounts (hundreds).
- **Tasks.** A task lives in exactly one project. Fields: title, free-text
  description, status (`todo`, `in_progress`, `done`), priority (`low`,
  `normal`, `high`, `urgent`), optional due date, optional assignee (must be a
  member of the project), timestamps. Projects at some client accounts have
  10,000+ tasks.
- **Comments.** A task has a threadless comment stream: author, body,
  timestamp. Comments can be edited and deleted by their author.

## Flows the API must support

1. Full create/read/update/delete for projects, tasks, and comments; invite,
   list, change role, and remove for members.
2. **Assign** a task to a member (and unassign it).
3. **Complete** a task (and reopen it if it was completed by mistake).
4. Every list view in the UI is backed by the API and must not fetch entire
   collections at once: the task list needs paging, filtering (by status,
   assignee, priority, due-before), and sorting (by due date, priority,
   creation time); project and member lists need paging too.

## Constraints

- All users authenticate; assume token-based auth handled by our gateway, but
  the API should still say what an unauthenticated or forbidden call gets.
- The web client team wants predictable, uniform responses across the whole
  API — they burned a quarter last year writing per-endpoint response parsers
  for our legacy internal API.
- Third parties will integrate against this API, so we cannot casually change
  it after launch.

## Deliverable

The API design as an OpenAPI 3.1 YAML document, plus whatever short notes you
think the reviewing engineers need.
