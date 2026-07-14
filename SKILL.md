---
name: devise-setup
description: >
  Installs and configures Devise on a Rails app with my preferred format: a
  single authenticatable model with UUIDs. Two patterns available — simple
  fixed roles with thin satellite models, or a multi-tenant Membership
  pattern (Account → Member/Admin → Membership → Tenant, e.g. Domain/Clinic/
  Organization). The authenticatable model has no fixed name — it can be
  `Account`, `User`, `Company`, `Commerce`, etc. depending on what the
  project needs. Trigger: "install devise", "configure devise", "add
  authentication", "setup auth with devise", "new devise installation".
---

Always reproduce this same pattern when installing Devise — but the name of
the authenticatable model, the pattern used, and any tenant/role names are
ALWAYS asked first, never assumed.

## Core idea of the pattern

- A single model is `devise`-authenticatable (it can be called `Account`,
  `User`, `Company`, `Commerce`, whatever the project needs).
- **Roles are optional**, and come in two possible shapes:
  - **Pattern A — simple roles**: the authenticatable model has a `role`
    enum (custom names, e.g. `user`/`admin`, or `owner`/`employee`) and an
    `after_create` callback that automatically creates the matching
    satellite record for that role (`create_<role>!`). Satellite models are
    thin: just `belongs_to :<resource>` plus attributes specific to that
    role. Use this when the project has no multi-tenancy — a satellite
    record belongs directly to the authenticatable model, nothing else.
  - **Pattern B — multi-tenant Membership**: the authenticatable model has a
    fixed `role` enum (`member`/`admin` — `admin` is the platform-level
    operator, `member` is a normal account). `Member` and `Admin` are its
    two possible satellites (`has_one`, auto-created via `after_create`,
    exactly like Pattern A but with these two fixed names). `Member` then
    joins a tenant model (`{{Tenant}}` — e.g. `Domain`, `Clinic`,
    `Organization`, `Workspace`; name asked in step 0) through a
    `Membership` model (`role`: `owner`/`member`, `status`:
    `active`/`inactive`). Use this whenever the project needs teams,
    workspaces, or several people sharing access to the same tenant
    resource.
  - If the project needs neither, the authenticatable model stays on its
    own, with no enum and no satellites.
- All tables use `id: :uuid`.
- No generated migration or class ever mentions "devise" in its name.
- Never create a brand-new migration file for a change that can be folded
  into a migration you just created in this same setup and haven't shipped
  yet (see "Folding migrations" below).

## Step 0 — ALWAYS ask before generating anything

Ask the user, for the current project, IN THIS ORDER:

1. **Which pattern?** None (no roles at all) / Pattern A (simple fixed
   roles) / Pattern B (multi-tenant Membership). Don't assume — Pattern B is
   more setup but is the right call the moment "teams", "workspaces",
   "several people access the same X", or "organizations/clinics/companies
   with staff" show up in the domain.
2. **Name of the authenticatable model** (singular, PascalCase). Examples:
   `Account`, `User`, `Company`, `Commerce`. Don't default to `Account`.
3. Depending on the pattern:
   - **Pattern A:** names of the roles / satellite models the authenticatable
     model auto-creates (e.g. `user`/`admin`, or `owner`/`staff`, or just
     one). These aren't fixed, they depend on the project's domain.
   - **Pattern B:** name of the tenant model (`{{Tenant}}` — e.g. `Domain`,
     `Clinic`, `Organization`, `Workspace`). The roles themselves
     (`member`/`admin` on the authenticatable model, `owner`/`member` on
     `Membership`) are fixed by convention and not asked.
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

For Pattern A, for each chosen role, `{{role}}` (snake_case singular, e.g.
`owner`) / `{{Role}}` (PascalCase, e.g. `Owner`) / `{{roles}}` (plural,
table, e.g. `owners`).

For Pattern B, for the tenant: `{{Tenant}}` (PascalCase singular, e.g.
`Clinic`) / `{{tenant}}` (snake_case singular, e.g. `clinic`) / `{{tenants}}`
(plural, e.g. `clinics`).

Substitute these placeholders in ALL of the following steps' code — don't
leave `Account`/`accounts` or `Domain`/`domains` hardcoded if the user asked
for different names.

## Common steps (all patterns)

### 1. Gemfile

```ruby
gem "devise"
```

```bash
bundle install
rails generate devise:install
```

Also check whether `dotenv-rails` is already in the Gemfile. If it isn't:

```ruby
gem "dotenv-rails"
```

```bash
bundle install
```

And create a `.env` file at the project root with at least:

```
PORT=3000
```

`config/environments/development.rb`'s `default_url_options` (step 3 below)
reads `ENV.fetch("PORT", 3000)` — without `dotenv-rails` and a `.env`, that
falls back silently to the hardcoded default, which works but isn't
configurable per machine.

### 2. Confirm UUID support

If the app doesn't generate UUIDs by default yet, add a migration that
enables the extension:

```ruby
enable_extension "pgcrypto"   # or "uuid-ossp" depending on the Postgres version
```

`gen_random_uuid()` requires Postgres 13+ (built-in) or the `pgcrypto`/`uuid-ossp`
extension. On Postgres 13+ this is often unnecessary — verify with
`psql --version` before assuming the extension migration is needed.

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
      t.string :role, null: false, default: "{{role_default}}" # only if using Pattern A or B

      t.timestamps null: false
    end

    add_index :{{resources}}, :email, unique: true
    add_index :{{resources}}, :reset_password_token, unique: true
    # add_index :{{resources}}, :confirmation_token, unique: true # only if you enabled Confirmable above
    # add_index :{{resources}}, :unlock_token, unique: true # only if you enabled Lockable above
  end
end
```

**Gotcha:** `rails generate devise {{Resource}}` writes the Trackable
columns **commented out** by default. Since the model's `devise(...)` call
always includes `:trackable` in this pattern, those columns must be
uncommented — a model declaring `:trackable` while its columns stay
commented out is a contradiction that breaks sign-in tracking silently
(no error, the columns just don't exist). Always uncomment them as part of
editing this migration, never leave the generator's default as-is.

### Folding migrations (don't create files you don't need)

If a later step in this same setup needs to add a column or a foreign key to
a table you already created a migration for — and that migration hasn't been
run in a previously shipped environment — **edit that migration file
directly instead of generating a new one.** Only reach for a separate
migration when altering a table whose creating migration has already been
migrated somewhere real (staging/production).

The recurring case in Pattern B: `Member` needs a `current_membership_id`
column that references `Membership`, but `Membership` doesn't exist yet when
`Member`'s table is created (ordering: members → tenant → memberships). Don't
solve this with a third `AddCurrentMembershipToMembers` migration. Instead:

- Add the plain column (no FK yet) inside the `create_members` migration.
- Add the foreign key constraint at the end of the `create_memberships`
  migration, once the `memberships` table actually exists:

```ruby
# db/migrate/..._create_members.rb
class CreateMembers < ActiveRecord::Migration[8.1]
  def change
    create_table :members, id: :uuid do |t|
      t.references :{{resource}}, null: false, foreign_key: true, type: :uuid
      t.uuid :current_membership_id

      t.timestamps
    end

    add_index :members, :current_membership_id
  end
end
```

```ruby
# db/migrate/..._create_memberships.rb
class CreateMemberships < ActiveRecord::Migration[8.1]
  def change
    create_table :memberships, id: :uuid do |t|
      t.references :member, null: false, foreign_key: true, type: :uuid
      t.references :{{tenant}}, null: false, foreign_key: true, type: :uuid
      t.string :role, null: false, default: "member"
      t.string :status, null: false, default: "active"

      t.timestamps
    end

    add_index :memberships, %i[member_id {{tenant}}_id], unique: true
    add_index :memberships, :{{tenant}}_id, unique: true, where: "role = 'owner'", name: "index_memberships_on_{{tenant}}_id_unique_owner"

    add_foreign_key :members, :memberships, column: :current_membership_id, on_delete: :nullify
  end
end
```

`on_delete: :nullify` is required here — without it, destroying a
`Membership` that happens to be someone's `current_membership` raises a
`ForeignKeyViolation` instead of just clearing the pointer.

### Last step — after everything is generated and migrated

Run the project's autocorrecting linter before considering the setup done:

```bash
bundle exec rubocop -A
```

## Pattern A: simple fixed roles (no tenant)

### A5. Satellite model migrations

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

### A6. Authenticatable model

**Without roles** (step 0 confirmed none of the patterns are used):

```ruby
class {{Resource}} < ApplicationRecord
  devise :database_authenticatable, :registerable, :recoverable, :rememberable, :validatable, :trackable

  validates :first_name, :last_name, presence: true
end
```

**With Pattern A roles:**

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

### A7. Satellite models

```ruby
class {{Role}} < ApplicationRecord
  belongs_to :{{resource}}
end
```

One per role agreed on in step 0. To add a new role later: create its
migration (`references :{{resource}}, type: :uuid`), its thin model with
`belongs_to :{{resource}}`, add it to the authenticatable model's `role` enum,
add its `has_one`, and the matching `when` in `create_associate_resource!`.

### A8. Controllers per role: namespace + dashboard

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

### A9. Routes

```ruby
devise_for :{{resources}}

namespace :{{role}} do
  root to: "dashboard#index"
end
# repeat the namespace for each role agreed on in step 0
```

If there are no roles, only the `devise_for :{{resources}}` line remains.

## Pattern B: multi-tenant Membership

### B5. Satellite, tenant, and join model migrations

Order matters — `{{Tenant}}` and `Member` must exist before `Membership`, and
`Membership` must exist before the `current_membership` foreign key can be
added (see "Folding migrations" above).

```ruby
# create_members
class CreateMembers < ActiveRecord::Migration[8.1]
  def change
    create_table :members, id: :uuid do |t|
      t.references :{{resource}}, null: false, foreign_key: true, type: :uuid
      t.uuid :current_membership_id

      t.timestamps
    end

    add_index :members, :current_membership_id
  end
end
```

```ruby
# create_admins
class CreateAdmins < ActiveRecord::Migration[8.1]
  def change
    create_table :admins, id: :uuid do |t|
      t.references :{{resource}}, null: false, foreign_key: true, type: :uuid
      t.timestamps
    end
  end
end
```

```ruby
# create_{{tenants}}
class Create{{Tenant}}s < ActiveRecord::Migration[8.1]
  def change
    create_table :{{tenants}}, id: :uuid do |t|
      t.string :name, null: false
      # any tenant-specific fields agreed on in step 0

      t.timestamps
    end
  end
end
```

```ruby
# create_memberships
class CreateMemberships < ActiveRecord::Migration[8.1]
  def change
    create_table :memberships, id: :uuid do |t|
      t.references :member, null: false, foreign_key: true, type: :uuid
      t.references :{{tenant}}, null: false, foreign_key: true, type: :uuid
      t.string :role, null: false, default: "member"
      t.string :status, null: false, default: "active"

      t.timestamps
    end

    add_index :memberships, %i[member_id {{tenant}}_id], unique: true
    add_index :memberships, :{{tenant}}_id, unique: true, where: "role = 'owner'", name: "index_memberships_on_{{tenant}}_id_unique_owner"

    add_foreign_key :members, :memberships, column: :current_membership_id, on_delete: :nullify
  end
end
```

```ruby
# create_membership_invites
class CreateMembershipInvites < ActiveRecord::Migration[8.1]
  def change
    create_table :membership_invites, id: :uuid do |t|
      t.references :{{tenant}}, null: false, foreign_key: true, type: :uuid
      t.references :invited_by, null: false, foreign_key: { to_table: :members }, type: :uuid
      t.string :email, null: false
      t.string :role, null: false, default: "member"
      t.string :token, null: false
      t.datetime :accepted_at
      t.datetime :expires_at, null: false

      t.timestamps
    end

    add_index :membership_invites, :token, unique: true
    add_index :membership_invites, %i[{{tenant}}_id email], unique: true, where: "accepted_at IS NULL", name: "index_membership_invites_on_{{tenant}}_id_and_email_when_pending"
  end
end
```

```bash
rails db:migrate
```

### B6. Authenticatable model

```ruby
class {{Resource}} < ApplicationRecord
  devise :database_authenticatable, :registerable, :recoverable, :rememberable, :validatable, :trackable

  has_one :admin
  has_one :member

  validates :first_name, :last_name, presence: true

  enum :role, { member: "member", admin: "admin" }, default: :member

  after_create :create_associate_resource!

  private

    def create_associate_resource!
      return create_admin! if admin?

      create_member!
    end
end
```

### B7. Member, Admin, Tenant, Membership, MembershipInvite models

```ruby
class Member < ApplicationRecord
  belongs_to :{{resource}}
  belongs_to :current_membership, class_name: "Membership", optional: true

  has_many :memberships, dependent: :destroy
  has_many :{{tenants}}, through: :memberships
end
```

```ruby
class Admin < ApplicationRecord
  belongs_to :{{resource}}
end
```

```ruby
class {{Tenant}} < ApplicationRecord
  has_one :owner_membership, -> { where(role: :owner) }, class_name: "Membership"
  has_one :owner, through: :owner_membership, source: :member

  has_many :memberships, dependent: :destroy
  has_many :members, through: :memberships
  has_many :membership_invites, dependent: :destroy

  validates :name, presence: true
end
```

```ruby
class Membership < ApplicationRecord
  belongs_to :member
  belongs_to :{{tenant}}

  enum :role, { owner: "owner", member: "member" }, default: :member
  enum :status, { active: "active", inactive: "inactive" }, default: :active

  validates :member_id, uniqueness: { scope: :{{tenant}}_id }
end
```

```ruby
class MembershipInvite < ApplicationRecord
  belongs_to :{{tenant}}
  belongs_to :invited_by, class_name: "Member"

  enum :role, { owner: "owner", member: "member" }, default: :member

  before_create :generate_token
  before_create :set_expires_at

  def accept!(member)
    transaction do
      {{tenant}}.memberships.create!(member: member, role: role)
      update!(accepted_at: Time.current)
    end
  end

  private

    def generate_token
      self.token = SecureRandom.hex(16)
    end

    def set_expires_at
      self.expires_at ||= 7.days.from_now
    end
end
```

Note `{{Resource}}` (e.g. `Account`) has no `dependent: :destroy` on its
`has_one :admin`/`has_one :member` — matching Devise's own philosophy of not
cascading destructively by default. Whoever deletes an `{{Resource}}` is
responsible for cleaning up its `member`/`admin` first.

### B8. Controllers: admin namespace + tenant-scoped namespace

```ruby
# app/controllers/admin/application_controller.rb
class Admin::ApplicationController < ApplicationController
  before_action :authenticate_{{resource}}!
  before_action :require_admin!

  private

    def require_admin!
      return if current_{{resource}}.admin?

      redirect_to root_path, alert: "You don't have permission to do that."
    end
end
```

```ruby
# app/controllers/admin/dashboard_controller.rb
class Admin::DashboardController < Admin::ApplicationController
  def index
  end
end
```

```ruby
# app/controllers/{{tenants}}/application_controller.rb
class {{Tenant}}s::ApplicationController < ApplicationController
  before_action :authenticate_{{resource}}!
  before_action :current_member!
  before_action :check_permission!

  private

    def current_member!
      @member = current_{{resource}}.member
    end

    def check_permission!
      return if current_{{resource}}.member?

      redirect_to root_path, alert: "You don't have permission to do that."
    end

    def set_current_{{tenant}}!
      @{{tenant}} = {{Tenant}}.find(params[:{{tenant}}_id])
    end

    def require_membership!
      @membership = @member.memberships.find_by({{tenant}}: @{{tenant}})

      return if @membership

      redirect_to root_path, alert: "You don't have permission to do that."
    end
end
```

```ruby
# app/controllers/{{tenants}}/dashboards_controller.rb
class {{Tenant}}s::DashboardsController < {{Tenant}}s::ApplicationController
  before_action :set_current_{{tenant}}!
  before_action :require_membership!

  def show
  end
end
```

Each namespace has its own `ApplicationController` — never share one across
`Admin` and `{{Tenant}}s`.

### B9. Routes

```ruby
devise_for :{{resources}}

namespace :admin do
  root to: "dashboard#index"
end

resources :{{tenants}}, only: [], module: "{{tenants}}", scope: "{{tenants}}" do
  resource :dashboard, only: :show
end
```

`resource :dashboard` (singular) is routed to a controller named
`DashboardsController` (plural) by Rails convention — the class must be
named accordingly (`{{Tenant}}s::DashboardsController`, not
`{{Tenant}}s::DashboardController`). Run `rails routes | grep {{tenant}}` to
confirm before moving on.

## Common closing steps (all patterns)

### 10. ApplicationController — permit extra params + redirect by role

```ruby
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?

  protected

    def after_sign_in_path_for(resource)
      return admin_root_path if resource.respond_to?(:admin?) && resource.admin?

      root_path
    end

    def configure_permitted_parameters
      devise_parameter_sanitizer.permit(:sign_in) do |{{resource}}_params|
        {{resource}}_params.permit(:email, :password)
      end

      devise_parameter_sanitizer.permit(:sign_up) do |{{resource}}_params|
        {{resource}}_params.permit(:first_name, :last_name, :email, :password) # add other extra fields from step 0
      end
    end
end
```

Skip the `after_sign_in_path_for` override entirely if the project uses no
roles at all (no `admin?` method to check).

### 11. Views (optional)

```bash
rails generate devise:views
```

Generates views under `app/views/devise/*`, ready to customize with the
project's design system.

### 12. Verify end to end

Before calling the setup done:

- `rails routes | grep -i "{{resource}}\|admin\|{{tenant}}"` — confirm every
  expected route exists.
- Create one authenticatable record per relevant role via `rails runner` (or
  console) and confirm the satellite/membership chain wires up correctly,
  then delete the test data.
- Boot the server and hit the root, sign-in, and any role-gated route to
  confirm redirects behave (unauthenticated → sign in; wrong role → alert +
  redirect).
- `bundle exec rubocop -A`.

## Quick checklist

- [ ] Asked and confirmed (step 0, in order): which pattern (none / A / B), authenticatable model name, role/tenant names, extra fields beyond first_name/last_name
- [ ] `gem "devise"` + `bundle install` + `rails generate devise:install`
- [ ] `dotenv-rails` present (added if missing) + `.env` with `PORT=3000`
- [ ] `default_url_options` configured in development and production (mailer)
- [ ] Migration renamed: no "devise" in the file or class name (`create_{{resources}}`, not `devise_create_{{resources}}`)
- [ ] `id: :uuid` on ALL new tables
- [ ] Trackable columns uncommented (the model declares `:trackable` — generator leaves them commented, must be fixed manually)
- [ ] Confirmable/Lockable columns commented out by default in the migration
- [ ] No migration created for something that could be folded into a migration from this same setup (see "Folding migrations")
- [ ] `{{Resource}}` = the sole authenticatable model
- [ ] Pattern A: `role` enum + `after_create :create_associate_resource!` on `{{Resource}}`, thin satellite models with `belongs_to :{{resource}}`, `{{Role}}::ApplicationController` + `{{Role}}::DashboardController` per role
- [ ] Pattern B: fixed `member`/`admin` role on `{{Resource}}`; `Member`/`Admin` satellites; `Membership` (fixed `owner`/`member` role, unique-owner-per-tenant partial index) joining `Member` to `{{Tenant}}`; `current_membership_id` on `Member` with `on_delete: :nullify`; `MembershipInvite`; `Admin::*` + `{{Tenant}}s::*` controller namespaces
- [ ] If NOT using any pattern: `{{Resource}}` stays simple, no enum, no satellites, no controller namespaces
- [ ] `devise_for :{{resources}}` in routes
- [ ] `configure_permitted_parameters` in `ApplicationController`
- [ ] Migrated and ran `rails db:migrate`
- [ ] Ran `bundle exec rubocop -A` as the last step

## What NOT to do

- Don't assume the authenticatable model is called `Account` — always ask (step 0).
- Don't assume the project needs a pattern, or which one — ask first, it's question #1 in step 0.
- Don't add a `role` enum or satellite models if the user said they won't be used.
- Don't make a satellite model (`User`, `Member`, etc.) the authenticatable one directly — it always goes through `{{Resource}}`.
- Don't put business logic specific to one role inside `{{Resource}}` — that lives in each satellite model.
- Don't use `id: :bigint` (the default) on new tables — the standard is UUID.
- Don't leave "devise" in the file/class name of any generated migration.
- Don't leave Trackable columns commented out — the model always declares `:trackable`, so the columns must exist. The generator comments them by default; uncomment as part of editing the migration.
- Don't enable Confirmable/Lockable columns by default in the migration — leave them commented out until needed.
- Don't forget ActionMailer's `default_url_options` if the project uses `:recoverable` or `:confirmable` — without it, emails break.
- Don't share one `ApplicationController` across roles or namespaces — each gets its own.
- Don't create a brand-new migration file for something that can be folded into an existing, not-yet-shipped migration from the same setup (e.g. don't add a separate `AddCurrentMembershipToMembers` migration — put the column in `create_members` and the FK in `create_memberships`).
- Don't add the `current_membership` foreign key without `on_delete: :nullify` — destroying a membership that's someone's current one will raise `ForeignKeyViolation` instead of clearing the pointer.
- Don't skip `bundle exec rubocop -A` at the end of the setup.
- Don't skip creating `.env` with `PORT=3000` when `dotenv-rails` had to be added — `default_url_options` silently falls back to a hardcoded default otherwise.
