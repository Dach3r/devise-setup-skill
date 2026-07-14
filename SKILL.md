---
name: devise-setup
description: >
  Installs and configures Devise on a Rails app with my preferred format: a
  single authenticatable model with UUIDs, a `role` enum, and thin satellite
  models auto-created via `after_create` based on the role. The
  authenticatable model has no fixed name — it can be `Account`, `User`,
  `Company`, `Commerce`, etc. depending on what the project needs. Trigger:
  "install devise", "configure devise", "add authentication", "setup auth
  with devise", "new devise installation".
---

Always reproduce this same pattern when installing Devise — but the name of
the authenticatable model and of the satellite roles are ALWAYS asked first,
never assumed.

## Core idea of the pattern

- A single model is `devise`-authenticatable (it can be called `Account`,
  `User`, `Company`, `Commerce`, whatever the project needs).
- **Roles are optional.** If the project needs them: the authenticatable
  model has a `role` enum (e.g. `user`/`admin`, or `owner`/`employee`) and an
  `after_create` callback that automatically creates the matching satellite
  record for that role (`create_<role>!`). Satellite models are thin: just
  `belongs_to :<resource>` plus attributes specific to that role. If the
  project does NOT need roles, the authenticatable model stays on its own,
  with no enum and no satellites.
- All tables use `id: :uuid`.
- No generated migration or class ever mentions "devise" in its name.

## Step 0 — ALWAYS ask before generating anything

Ask the user, for the current project, IN THIS ORDER:

1. **Is it going to use roles / satellite models or not?** This decides
   whether steps 5, 7, and 8 happen at all, and which shape step 6 takes.
   Don't assume yes just because it's the most common case.
2. **Name of the authenticatable model** (singular, PascalCase). Examples:
   `Account`, `User`, `Company`, `Commerce`. Don't default to `Account`.
3. **Only if they answered yes to roles:** names of the roles / satellite
   models that model is going to auto-create (e.g. `user`/`admin`, or
   `owner`/`staff`, or just one). These aren't fixed, they depend on the
   project's domain.
4. **Extra fields.** `first_name` and `last_name` are always included as a
   base. Ask what other extra fields the authenticatable model needs besides
   those two.

With those answers, define the three forms of the name to use throughout the
rest of the guide:

| Placeholder | Form | Example if the user picked "Company" |
|---|---|---|
| `{{Resource}}` | PascalCase singular (class) | `Company` |
| `{{resource}}` | snake_case singular (associations, variables) | `company` |
| `{{resources}}` | snake_case plural (table, routes) | `companies` |

And for each chosen role, `{{role}}` (snake_case singular, e.g. `owner`) /
`{{Role}}` (PascalCase, e.g. `Owner`) / `{{roles}}` (plural, table, e.g. `owners`).

Substitute these placeholders in ALL of the following steps' code — don't
leave `Account`/`accounts` hardcoded if the user asked for a different name.

## Steps

### 1. Gemfile

```ruby
gem "devise"
```

```bash
bundle install
rails generate devise:install
```

### 2. Confirm UUID support

If the app doesn't generate UUIDs by default yet, add a migration that
enables the extension:

```ruby
enable_extension "pgcrypto"   # or "uuid-ossp" depending on the Postgres version
```

`gen_random_uuid()` requires Postgres 13+ (built-in) or the `pgcrypto`/`uuid-ossp`
extension.

### 3. Configure ActionMailer (required by :recoverable and :confirmable)

Devise needs `default_url_options` to build the links in its emails (reset
password, confirmation). Without this, those flows break when generating the
URL.

```ruby
# config/environments/development.rb
config.action_mailer.default_url_options = { host: "localhost", port: ENV.fetch("PORT", 3000) }
```

```ruby
# config/environments/production.rb
config.action_mailer.default_url_options = { host: ENV.fetch("APP_HOST") }
```

### 4. Generate the authenticatable model with Devise

```bash
rails generate devise {{Resource}}
```

This generates a migration named `..._devise_create_{{resources}}.rb` with
the class `DeviseCreate{{Resource}}`. Never leave "devise" in the name —
rename the file and the class:

```bash
mv db/migrate/*_devise_create_{{resources}}.rb db/migrate/$(date +%Y%m%d%H%M%S)_create_{{resources}}.rb
```

And inside the file, change `class DeviseCreate{{Resource}}` → `class Create{{Resource}}`.

Edit the generated migration so it ends up with `id: :uuid` and add the extra
fields agreed on in step 0:

```ruby
class Create{{Resource}} < ActiveRecord::Migration[8.1]
  def change
    create_table :{{resources}}, id: :uuid do |t|
      ## Database authenticatable
      t.string :email, null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Trackable
      t.integer  :sign_in_count, default: 0, null: false
      t.datetime :current_sign_in_at
      t.datetime :last_sign_in_at
      t.string   :current_sign_in_ip
      t.string   :last_sign_in_ip

      ## Confirmable — inactive by default, uncomment + add :confirmable to the model if needed
      # t.string   :confirmation_token
      # t.datetime :confirmed_at
      # t.datetime :confirmation_sent_at
      # t.string   :unconfirmed_email

      ## Lockable — inactive by default, uncomment + add :lockable to the model if needed
      # t.integer  :failed_attempts, default: 0, null: false
      # t.string   :unlock_token
      # t.datetime :locked_at

      ## Extra fields (first_name/last_name always included; add the rest agreed on in step 0)
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :role, null: false, default: "{{role_default}}" # only if step 0 confirmed roles

      t.timestamps null: false
    end

    add_index :{{resources}}, :email, unique: true
    add_index :{{resources}}, :reset_password_token, unique: true
    # add_index :{{resources}}, :confirmation_token, unique: true # only if you enabled Confirmable above
    # add_index :{{resources}}, :unlock_token, unique: true # only if you enabled Lockable above
  end
end
```

### 5. Satellite model migrations (ONLY if step 0 confirmed roles; otherwise skip to step 9)

```ruby
class Create{{Role}} < ActiveRecord::Migration[8.1]
  def change
    create_table :{{roles}}, id: :uuid do |t|
      t.references :{{resource}}, null: false, foreign_key: true, type: :uuid
      t.timestamps
    end
  end
end
```

Repeat for each role agreed on in step 0.

```bash
rails db:migrate
```

### 6. Authenticatable model

**Without roles** (step 0 confirmed they're not used):

```ruby
class {{Resource}} < ApplicationRecord
  devise :database_authenticatable, :registerable, :recoverable, :rememberable, :validatable, :trackable

  validates :first_name, :last_name, presence: true
end
```

**With roles** (step 0 confirmed they're used):

```ruby
class {{Resource}} < ApplicationRecord
  devise :database_authenticatable, :registerable, :recoverable, :rememberable, :validatable, :trackable

  has_one :{{role_1}}
  has_one :{{role_2}}
  # ... one has_one per role agreed on in step 0

  validates :first_name, :last_name, presence: true

  enum :role, { {{role_1}}: "{{role_1}}", {{role_2}}: "{{role_2}}" }, default: :{{role_default}}

  after_create :create_associate_resource!

  private

  def create_associate_resource!
    case role
    when "{{role_1}}" then create_{{role_1}}!
    when "{{role_2}}" then create_{{role_2}}!
    end
  end
end
```

Adjust the `devise` modules to fit the real needs (add `:confirmable`,
`:lockable`, etc. if the migration included them).

### 7. Satellite models (ONLY if step 0 confirmed roles)

```ruby
class {{Role}} < ApplicationRecord
  belongs_to :{{resource}}
end
```

One per role agreed on in step 0. To add a new role later: create its
migration (`references :{{resource}}, type: :uuid`), its thin model with
`belongs_to :{{resource}}`, add it to the authenticatable model's `role` enum,
add its `has_one`, and the matching `when` in `create_associate_resource!`.

### 8. Controllers per role: namespace + dashboard (ONLY if step 0 confirmed roles)

For each role, its own controller namespace with an `ApplicationController`
that inherits from the global `ApplicationController` and enforces the role,
plus a default `DashboardController` — the same format as a classic Rails
admin namespace (`Admin::ApplicationController` + `Admin::DashboardController`
with `root to: "dashboard#index"` inside its own namespace).

```ruby
# app/controllers/{{role}}/application_controller.rb
class {{Role}}::ApplicationController < ApplicationController
  before_action :authenticate_{{resource}}!
  before_action :require_{{role}}!

  private

    def require_{{role}}!
      return if current_{{resource}}.{{role}}?

      redirect_to root_path, alert: "You don't have permission to do that."
    end
end
```

```ruby
# app/controllers/{{role}}/dashboard_controller.rb
class {{Role}}::DashboardController < {{Role}}::ApplicationController
  def index
  end
end
```

Repeat for each role agreed on in step 0 — each namespace has its own
`ApplicationController`, they don't share one across roles.

### 9. Routes

```ruby
devise_for :{{resources}}

namespace :{{role}} do
  root to: "dashboard#index"
end
# repeat the namespace for each role agreed on in step 0
```

If there are no roles, only the `devise_for :{{resources}}` line remains.

### 10. ApplicationController — permit extra params

```ruby
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_in) do |{{resource}}_params|
      {{resource}}_params.permit(:email, :password)
    end

    devise_parameter_sanitizer.permit(:sign_up) do |{{resource}}_params|
      {{resource}}_params.permit(:first_name, :last_name, :email, :password)
    end
  end
end
```

### 11. Views (optional)

```bash
rails generate devise:views
```

Generates views under `app/views/devise/*`, ready to customize with the
project's design system.

## Quick checklist

- [ ] Asked and confirmed (step 0, in order): uses roles?, authenticatable model name, satellite roles (if any), extra fields beyond first_name/last_name
- [ ] `gem "devise"` + `bundle install` + `rails generate devise:install`
- [ ] `default_url_options` configured in development and production (mailer)
- [ ] Migration renamed: no "devise" in the file or class name (`create_{{resources}}`, not `devise_create_{{resources}}`)
- [ ] `id: :uuid` on ALL new tables
- [ ] Confirmable/Lockable columns commented out by default in the migration
- [ ] `{{Resource}}` = the sole authenticatable model
- [ ] If using roles: `role` enum + `after_create :create_associate_resource!` on `{{Resource}}`, thin satellite models with `belongs_to :{{resource}}`
- [ ] If using roles: `{{Role}}::ApplicationController` + `{{Role}}::DashboardController` namespace per role, with its `root` in the routes namespace
- [ ] If NOT using roles: `{{Resource}}` stays simple, no enum, no satellites, no controller namespaces
- [ ] `devise_for :{{resources}}` in routes
- [ ] `configure_permitted_parameters` in `ApplicationController`
- [ ] Migrated and ran `rails db:migrate`

## What NOT to do

- Don't assume the authenticatable model is called `Account` — always ask (step 0).
- Don't assume the project uses roles — ask first, it's question #1 in step 0.
- Don't add a `role` enum or satellite models if the user said they won't be used.
- Don't make a satellite model (`User`, etc.) the authenticatable one directly — it always goes through `{{Resource}}`.
- Don't put business logic specific to one role inside `{{Resource}}` — that lives in each satellite model.
- Don't use `id: :bigint` (the default) on new tables — the standard is UUID.
- Don't leave "devise" in the file/class name of any generated migration.
- Don't enable Confirmable/Lockable columns by default in the migration — leave them commented out until needed.
- Don't forget ActionMailer's `default_url_options` if the project uses `:recoverable` or `:confirmable` — without it, emails break.
- Don't share one `ApplicationController` across roles — each role gets its own, in its own namespace.
