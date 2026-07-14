# devise-setup

A [Claude Code](https://claude.com/claude-code) skill that installs and
configures Devise on a Rails app following a specific opinionated pattern:

- A single `devise`-authenticatable model (name is not fixed — `Account`,
  `User`, `Company`, `Commerce`, whatever the project calls it).
- Roles are optional. When used, the authenticatable model has a `role` enum
  and an `after_create` callback that automatically creates the matching thin
  satellite model (`User`, `Admin`, or whatever roles the project needs).
- All tables use `id: :uuid`.
- No generated migration or class ever has "devise" in its name.

See [`SKILL.md`](./SKILL.md) for the full step-by-step guide the skill
follows.

## Install

Clone this repo, then symlink it into your personal skills directory so
Claude Code can discover it:

```bash
git clone git@github.com:Dach3r/devise-setup-skill.git
ln -s "$(pwd)/devise-setup-skill" ~/.claude/skills/devise-setup
```

Restart Claude Code (or start a new session) so it picks up the new skill.

## Usage

From any Rails project, ask Claude Code to install Devise, e.g.:

```
instala devise
configura devise
setup auth con devise
```

The skill always starts by asking a few questions before generating anything:

1. Whether the project needs roles / satellite models at all.
2. The name of the authenticatable model (`Account`, `User`, `Company`, ...).
3. If roles are needed, their names (`user`/`admin`, `owner`/`staff`, ...).
4. Any extra fields beyond the default `first_name`/`last_name`.

It then walks through installing the gem, generating and renaming the
migration, wiring up UUIDs, ActionMailer config, the models, the
role-namespaced controllers (if any), routes, and `ApplicationController`
param sanitization — all following the format above.

## Updating

Edit [`SKILL.md`](./SKILL.md) and push — since it's symlinked, every project
picks up the change immediately without reinstalling.
