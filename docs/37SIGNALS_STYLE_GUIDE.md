# The 37signals/DHH Rails Style Guide

A comprehensive guide to application design based on deep analysis of the Fizzy codebase. This document captures the patterns, philosophies, and deliberate omissions that define the 37signals approach to Rails development.

---

## Table of Contents

1. [Philosophy Overview](#philosophy-overview)
2. [Dependencies & What's Notably Absent](#dependencies--whats-notably-absent)
3. [Routing: Everything is CRUD](#routing-everything-is-crud)
4. [Controller Design](#controller-design)
5. [Controller Concerns: The Complete Catalog](#controller-concerns-the-complete-catalog)
6. [Model Layer & Concerns](#model-layer--concerns)
7. [Authentication Without Devise](#authentication-without-devise)
8. [State as Records, Not Booleans](#state-as-records-not-booleans)
9. [Views & Turbo/Hotwire Patterns](#views--turbohotwire-patterns)
10. [Background Jobs](#background-jobs)
11. [Testing Approach](#testing-approach)
12. [What They Deliberately Avoid](#what-they-deliberately-avoid)
13. [Naming Conventions](#naming-conventions)
14. [Product Design Inferences](#product-design-inferences)

---

## Philosophy Overview

The 37signals approach can be summarized as: **"Vanilla Rails is plenty."** They maximize what Rails gives you out of the box, minimize dependencies, and resist abstractions until absolutely necessary.

Core principles:
- **Rich domain models** over service objects
- **CRUD controllers** over custom actions
- **Concerns** for horizontal code sharing
- **Records as state** over boolean columns
- **Database-backed everything** (no Redis)
- **Build it yourself** before reaching for gems

---

## Dependencies & What's Notably Absent

### What They Use

```ruby
# Gemfile

# Core Rails (running edge!)
gem "rails", github: "rails/rails", branch: "main"

# Their Hotwire stack
gem "turbo-rails"
gem "stimulus-rails"
gem "importmap-rails"
gem "propshaft"

# Database-backed infrastructure (NO Redis!)
gem "solid_queue"    # Jobs
gem "solid_cache"    # Caching
gem "solid_cable"    # WebSockets

# Their own gems
gem "geared_pagination"
gem "lexxy"          # Rich text
gem "mittens"        # Email

# Minimal, focused gems
gem "bcrypt"         # Password hashing
gem "rqrcode"        # QR codes
gem "redcarpet"      # Markdown
```

### What's Notably ABSENT

**DO NOT USE:**

| Gem/Pattern | Why They Avoid It |
|-------------|-------------------|
| `devise` | Auth is ~150 lines of custom code. Devise is overkill. |
| `pundit`/`cancancan` | Authorization lives in models (`can_administer_card?`) |
| `dry-rb` gems | Over-engineered for most Rails apps |
| `interactor`/`command` | Service objects are rarely needed |
| `view_component` | ERB partials are fine |
| `sidekiq` | Solid Queue uses the database (no Redis) |
| `redis` | Database-backed everything |
| `elasticsearch` | Custom sharded MySQL full-text search |
| `graphql` | REST with Turbo is sufficient |
| `rspec` | Minitest is simpler and faster |

---

## Routing: Everything is CRUD

### The Core Principle

Every action maps to a CRUD verb. When something doesn't fit, **create a new resource**.

```ruby
# BAD: Custom actions on existing resource
resources :cards do
  post :close
  post :reopen
  post :archive
  post :gild
end

# GOOD: New resources for each state change
resources :cards do
  resource :closure      # POST to close, DELETE to reopen
  resource :goldness     # POST to gild, DELETE to ungild
  resource :not_now      # POST to postpone
  resource :pin          # POST to pin, DELETE to unpin
  resource :watch        # POST to watch, DELETE to unwatch
end
```

### Real Examples from Fizzy Routes

```ruby
# config/routes.rb

resources :cards do
  scope module: :cards do
    resource :board           # Moving card to different board
    resource :closure         # Closing/reopening
    resource :column          # Assigning to workflow column
    resource :goldness        # Highlighting as important
    resource :image           # Managing header image
    resource :not_now         # Postponing
    resource :pin             # Pinning to sidebar
    resource :publish         # Publishing draft
    resource :reading         # Marking as read
    resource :triage          # Triaging
    resource :watch           # Subscribing to updates

    resources :assignments    # Managing assignees
    resources :steps          # Checklist items
    resources :taggings       # Tags
    resources :comments do
      resources :reactions    # Emoji reactions
    end
  end
end
```

### Namespace for Context

```ruby
# Board-specific resources
resources :boards do
  scope module: :boards do
    resource :publication    # Publishing publicly
    resource :entropy        # Auto-postpone settings
    resource :involvement    # User's involvement level

    namespace :columns do
      resource :not_now      # "Not Now" pseudo-column
      resource :stream       # Main stream view
      resource :closed       # Closed cards view
    end
  end
end
```

### Use `resolve` for Custom URL Generation

```ruby
# Make polymorphic_url work correctly for nested resources
resolve "Comment" do |comment, options|
  options[:anchor] = ActionView::RecordIdentifier.dom_id(comment)
  route_for :card, comment.card, options
end

resolve "Notification" do |notification, options|
  polymorphic_url(notification.notifiable_target, options)
end
```

---

## Controller Design

### Thin Controllers, Rich Models

Controllers should be thin orchestrators. Business logic lives in models.

```ruby
# GOOD: Controller just orchestrates
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    @card.close  # All logic in model

    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end

  def destroy
    @card.reopen  # All logic in model

    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end
end
```

```ruby
# BAD: Business logic in controller
class Cards::ClosuresController < ApplicationController
  def create
    @card.transaction do
      @card.create_closure!(user: Current.user)
      @card.events.create!(action: :closed, creator: Current.user)
      @card.watchers.each { |w| NotificationMailer.card_closed(w, @card).deliver_later }
    end
  end
end
```

### Concerns for Shared Controller Behavior

```ruby
# app/controllers/concerns/card_scoped.rb
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card, :set_board
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find_by!(number: params[:card_id])
    end

    def set_board
      @board = @card.board
    end

    def render_card_replacement
      render turbo_stream: turbo_stream.replace(
        [@card, :card_container],
        partial: "cards/container",
        method: :morph,
        locals: { card: @card.reload }
      )
    end
end
```

### ApplicationController is Minimal

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Authentication
  include Authorization
  include BlockSearchEngineIndexing
  include CurrentRequest, CurrentTimezone, SetPlatform
  include RequestForgeryProtection
  include TurboFlash, ViewTransitions
  include RoutingHeaders

  etag { "v1" }
  stale_when_importmap_changes
  allow_browser versions: :modern
end
```

### Authorization in Controller, Permission Logic in Model

```ruby
# Controller checks permission
class CardsController < ApplicationController
  before_action :ensure_permission_to_administer_card, only: [:destroy]

  private
    def ensure_permission_to_administer_card
      head :forbidden unless Current.user.can_administer_card?(@card)
    end
end

# Model defines what permission means
class User < ApplicationRecord
  def can_administer_card?(card)
    admin? || card.creator == self
  end

  def can_administer_board?(board)
    admin? || board.creator == self
  end
end
```

---

## Controller Concerns: The Complete Catalog

Controller concerns are the secret sauce of 37signals controllers. They create a vocabulary of reusable behaviors that compose beautifully. Here's every concern and when to use it.

### Resource Scoping Concerns

These concerns handle loading parent resources for nested controllers.

#### CardScoped - For Card Sub-resources

```ruby
# app/controllers/concerns/card_scoped.rb
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card, :set_board
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find_by!(number: params[:card_id])
    end

    def set_board
      @board = @card.board
    end

    def render_card_replacement
      render turbo_stream: turbo_stream.replace(
        [@card, :card_container],
        partial: "cards/container",
        method: :morph,
        locals: { card: @card.reload }
      )
    end
end
```

**Usage Pattern:**

```ruby
# Any controller nested under cards uses this
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    @card.close
    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end
end

class Cards::WatchesController < ApplicationController
  include CardScoped

  def create
    @card.watch_by Current.user
    # ...
  end
end

class Cards::PinsController < ApplicationController
  include CardScoped

  def create
    @pin = @card.pin_by Current.user
    # ...
  end
end
```

**Key insight:** The concern provides `render_card_replacement` - a shared way to update the card UI. This is critical for consistency across all card actions.

#### BoardScoped - For Board Sub-resources

```ruby
# app/controllers/concerns/board_scoped.rb
module BoardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_board
  end

  private
    def set_board
      @board = Current.user.boards.find(params[:board_id])
    end

    def ensure_permission_to_admin_board
      unless Current.user.can_administer_board?(@board)
        head :forbidden
      end
    end
end
```

**Usage:** Any controller under `boards/` uses this.

```ruby
class Boards::ColumnsController < ApplicationController
  include BoardScoped

  def create
    @column = @board.columns.create!(column_params)
    # ...
  end
end

class Boards::PublicationsController < ApplicationController
  include BoardScoped
  before_action :ensure_permission_to_admin_board

  def create
    @board.publish
  end
end
```

#### ColumnScoped - For Column Sub-resources

```ruby
# app/controllers/concerns/column_scoped.rb
module ColumnScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_column
  end

  private
    def set_column
      @column = Current.user.accessible_columns.find(params[:column_id])
    end
end
```

**Usage:**

```ruby
class Columns::LeftPositionsController < ApplicationController
  include ColumnScoped

  def create
    @left_column = @column.left_column
    @column.move_left
  end
end
```

### Request Context Concerns

These concerns capture and propagate request context.

#### CurrentRequest - Populate Current with Request Data

```ruby
# app/controllers/concerns/current_request.rb
module CurrentRequest
  extend ActiveSupport::Concern

  included do
    before_action do
      Current.http_method = request.method
      Current.request_id  = request.uuid
      Current.user_agent  = request.user_agent
      Current.ip_address  = request.ip
      Current.referrer    = request.referrer
    end
  end
end
```

**Why this matters:** Models and jobs can access request context via `Current` without parameter passing. For example, logging who created something:

```ruby
class Signup
  def create_identity
    Identity.create!(
      email_address: email_address,
      # These come from Current, not parameters!
      ip_address: Current.ip_address,
      user_agent: Current.user_agent
    )
  end
end
```

#### CurrentTimezone - User Timezone from Cookie

```ruby
# app/controllers/concerns/current_timezone.rb
# FIXME: This should move upstream to Rails. It's a good pattern.
module CurrentTimezone
  extend ActiveSupport::Concern

  included do
    around_action :set_current_timezone
    helper_method :timezone_from_cookie
    etag { timezone_from_cookie }
  end

  private
    def set_current_timezone(&)
      Time.use_zone(timezone_from_cookie, &)
    end

    def timezone_from_cookie
      @timezone_from_cookie ||= begin
        timezone = cookies[:timezone]
        ActiveSupport::TimeZone[timezone] if timezone.present?
      end
    end
end
```

**Key patterns:**
1. `around_action` wraps the entire request in the user's timezone
2. `etag` includes timezone - different timezones get different cached responses
3. `helper_method` makes it available in views
4. Cookie is set client-side by JavaScript detecting the user's timezone

#### SetPlatform - Detect Mobile/Desktop

```ruby
# app/controllers/concerns/set_platform.rb
module SetPlatform
  extend ActiveSupport::Concern

  included do
    helper_method :platform
  end

  private
    def platform
      @platform ||= ApplicationPlatform.new(request.user_agent)
    end
end
```

**Usage in views:**

```erb
<% if platform.mobile? %>
  <%= render "mobile_nav" %>
<% else %>
  <%= render "desktop_nav" %>
<% end %>
```

### Filtering & Pagination Concerns

#### FilterScoped - Complex Filtering

```ruby
# app/controllers/concerns/filter_scoped.rb
module FilterScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_filter
    before_action :set_user_filtering
  end

  private
    def set_filter
      if params[:filter_id].present?
        @filter = Current.user.filters.find(params[:filter_id])
      else
        @filter = Current.user.filters.from_params(filter_params)
      end
    end

    def filter_params
      params.reverse_merge(**Filter.default_values)
            .permit(*Filter::PERMITTED_PARAMS)
    end

    def set_user_filtering
      @user_filtering = User::Filtering.new(Current.user, @filter, expanded: expanded_param)
    end
end
```

**The Filter model does the heavy lifting:**

```ruby
class Filter < ApplicationRecord
  def cards
    result = creator.accessible_cards.preloaded.published
    result = result.indexed_by(indexed_by)
    result = result.sorted_by(sorted_by)
    result = result.where(board: boards.ids) if boards.present?
    result = result.tagged_with(tags.ids) if tags.present?
    result = result.assigned_to(assignees.ids) if assignees.present?
    # ... more filtering
    result.distinct
  end
end
```

**Pattern:** Filters are persisted! Users can save and name their filters.

### Security & Headers Concerns

#### BlockSearchEngineIndexing - Prevent Crawling

```ruby
# app/controllers/concerns/block_search_engine_indexing.rb
module BlockSearchEngineIndexing
  extend ActiveSupport::Concern

  included do
    after_action :block_search_engine_indexing
  end

  private
    def block_search_engine_indexing
      headers["X-Robots-Tag"] = "none"
    end
end
```

**Why:** Private app content shouldn't appear in search results, even if someone links to it.

#### RequestForgeryProtection - Modern CSRF

```ruby
# app/controllers/concerns/request_forgery_protection.rb
module RequestForgeryProtection
  extend ActiveSupport::Concern

  included do
    after_action :append_sec_fetch_site_to_vary_header
  end

  private
    def append_sec_fetch_site_to_vary_header
      vary_header = response.headers["Vary"].to_s.split(",").map(&:strip).reject(&:blank?)
      response.headers["Vary"] = (vary_header + ["Sec-Fetch-Site"]).join(",")
    end

    def verified_request?
      request.get? || request.head? || !protect_against_forgery? ||
        (valid_request_origin? && safe_fetch_site?)
    end

    SAFE_FETCH_SITES = %w[same-origin same-site]

    def safe_fetch_site?
      SAFE_FETCH_SITES.include?(sec_fetch_site_value) ||
        (sec_fetch_site_value.nil? && api_request?)
    end

    def api_request?
      request.format.json?
    end
end
```

**Modern approach:** Uses `Sec-Fetch-Site` header instead of tokens. Browsers set this automatically - can't be spoofed by JavaScript.

### Turbo/View Concerns

#### TurboFlash - Flash Messages via Turbo Stream

```ruby
# app/controllers/concerns/turbo_flash.rb
module TurboFlash
  extend ActiveSupport::Concern

  included do
    helper_method :turbo_stream_flash
  end

  private
    def turbo_stream_flash(**flash_options)
      turbo_stream.replace(:flash, partial: "layouts/shared/flash", locals: { flash: flash_options })
    end
end
```

**Usage in controller:**

```ruby
def create
  @comment = @card.comments.create!(comment_params)

  respond_to do |format|
    format.turbo_stream do
      render turbo_stream: [
        turbo_stream.append(:comments, @comment),
        turbo_stream_flash(notice: "Comment added!")
      ]
    end
  end
end
```

#### ViewTransitions - Disable on Refresh

```ruby
# app/controllers/concerns/view_transitions.rb
# FIXME: Upstream this fix to turbo-rails
module ViewTransitions
  extend ActiveSupport::Concern

  included do
    before_action :disable_view_transitions, if: :page_refresh?
  end

  private
    def disable_view_transitions
      @disable_view_transition = true
    end

    def page_refresh?
      request.referrer.present? && request.referrer == request.url
    end
end
```

**Why:** View transitions on page refresh look weird. This disables them automatically.

### Composing Concerns: Real Controllers

Here's how these concerns compose in practice:

```ruby
# A full-featured nested controller
class Cards::AssignmentsController < ApplicationController
  include CardScoped  # Gets @card, @board, render_card_replacement

  def new
    @assigned_to = @card.assignees.active.alphabetically.where.not(id: Current.user)
    @users = @board.users.active.alphabetically.where.not(id: @card.assignees)
    fresh_when etag: [@users, @card.assignees]  # HTTP caching!
  end

  def create
    @card.toggle_assignment @board.users.active.find(params[:assignee_id])

    respond_to do |format|
      format.turbo_stream
      format.json { head :no_content }
    end
  end
end
```

```ruby
# A timeline controller composing multiple concerns
class Events::Days::ColumnsController < ApplicationController
  include DayTimelinesScoped  # Which includes FilterScoped

  def show
    @column = @board.columns.find(params[:id])
  end
end
```

### Concern Composition Rules

1. **Concerns can include other concerns:**
   ```ruby
   module DayTimelinesScoped
     include FilterScoped  # Inherits all of FilterScoped
     # ...
   end
   ```

2. **Use `before_action` in `included` block:**
   ```ruby
   included do
     before_action :set_card
   end
   ```

3. **Provide shared private methods:**
   ```ruby
   def render_card_replacement
     # Reusable across all CardScoped controllers
   end
   ```

4. **Use `helper_method` for view access:**
   ```ruby
   included do
     helper_method :platform, :timezone_from_cookie
   end
   ```

5. **Add to `etag` for HTTP caching:**
   ```ruby
   included do
     etag { timezone_from_cookie }
   end
   ```

---

## Model Layer & Concerns

### Heavy Use of Concerns for Horizontal Behavior

Models include many concerns, each handling one aspect:

```ruby
# app/models/card.rb
class Card < ApplicationRecord
  include Assignable, Attachments, Broadcastable, Closeable, Colored,
    Entropic, Eventable, Exportable, Golden, Mentions, Multistep,
    Pinnable, Postponable, Promptable, Readable, Searchable, Stallable,
    Statuses, Storage::Tracked, Taggable, Triageable, Watchable

  belongs_to :account, default: -> { board.account }
  belongs_to :board
  belongs_to :creator, class_name: "User", default: -> { Current.user }

  has_many :comments, dependent: :destroy
  has_one_attached :image, dependent: :purge_later
  has_rich_text :description

  # Minimal model code - behavior is in concerns
end
```

### Concern Structure: Self-Contained Behavior

Each concern is self-contained with associations, scopes, and methods:

```ruby
# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
    scope :recently_closed_first, -> { closed.order("closures.created_at": :desc) }
  end

  def closed?
    closure.present?
  end

  def open?
    !closed?
  end

  def closed_by
    closure&.user
  end

  def close(user: Current.user)
    unless closed?
      transaction do
        create_closure! user: user
        track_event :closed, creator: user
      end
    end
  end

  def reopen(user: Current.user)
    if closed?
      transaction do
        closure&.destroy
        track_event :reopened, creator: user
      end
    end
  end
end
```

### Default Values via Lambdas

```ruby
class Card < ApplicationRecord
  belongs_to :account, default: -> { board.account }
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end

class Comment < ApplicationRecord
  belongs_to :account, default: -> { card.account }
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end
```

### Current for Request Context

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :identity, :account
  attribute :http_method, :request_id, :user_agent, :ip_address, :referrer

  def session=(value)
    super(value)
    self.identity = session.identity if value.present?
  end

  def identity=(identity)
    super(identity)
    self.user = identity.users.find_by(account: account) if identity.present?
  end
end
```

---

## Authentication Without Devise

### Custom Passwordless Magic Link Auth (~150 lines)

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_account
    before_action :require_authentication
    helper_method :authenticated?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
      before_action :resume_session, **options
    end
  end

  private
    def authenticated?
      Current.identity.present?
    end

    def require_authentication
      resume_session || authenticate_by_bearer_token || request_authentication
    end

    def resume_session
      if session = find_session_by_cookie
        set_current_session session
      end
    end

    def find_session_by_cookie
      Session.find_signed(cookies.signed[:session_token])
    end

    def start_new_session_for(identity)
      identity.sessions.create!(
        user_agent: request.user_agent,
        ip_address: request.remote_ip
      ).tap { |session| set_current_session session }
    end

    def set_current_session(session)
      Current.session = session
      cookies.signed.permanent[:session_token] = {
        value: session.signed_id,
        httponly: true,
        same_site: :lax
      }
    end
end
```

### Simple Session Model

```ruby
# app/models/session.rb
class Session < ApplicationRecord
  belongs_to :identity
end
```

### Magic Link Model

```ruby
# app/models/magic_link.rb
class MagicLink < ApplicationRecord
  CODE_LENGTH = 6
  EXPIRATION_TIME = 15.minutes

  belongs_to :identity

  enum :purpose, %w[sign_in sign_up], prefix: :for, default: :sign_in

  scope :active, -> { where(expires_at: Time.current...) }
  scope :stale, -> { where(expires_at: ..Time.current) }

  before_validation :generate_code, on: :create
  before_validation :set_expiration, on: :create

  def self.consume(code)
    active.find_by(code: Code.sanitize(code))&.consume
  end

  def consume
    destroy
    self
  end
end
```

---

## State as Records, Not Booleans

### The Pattern

Instead of `closed: boolean`, create a separate record. This gives you:
- Timestamp of when it happened
- Who did it
- Easy scoping via `joins` and `where.missing`

```ruby
# BAD: Boolean column
class Card < ApplicationRecord
  # closed: boolean column in cards table

  scope :closed, -> { where(closed: true) }
  scope :open, -> { where(closed: false) }
end

# GOOD: Separate record
class Closure < ApplicationRecord
  belongs_to :card, touch: true
  belongs_to :user, optional: true
  # created_at gives you when
  # user gives you who
end

class Card < ApplicationRecord
  has_one :closure, dependent: :destroy

  scope :closed, -> { joins(:closure) }
  scope :open, -> { where.missing(:closure) }

  def closed?
    closure.present?
  end
end
```

### Real Examples

```ruby
# Closure - tracks when/who closed a card
class Closure < ApplicationRecord
  belongs_to :account, default: -> { card.account }
  belongs_to :card, touch: true
  belongs_to :user, optional: true
end

# Goldness - marks a card as "golden" (important)
class Card::Goldness < ApplicationRecord
  belongs_to :account, default: -> { card.account }
  belongs_to :card, touch: true
end

# NotNow - marks a card as postponed
class Card::NotNow < ApplicationRecord
  belongs_to :account, default: -> { card.account }
  belongs_to :card, touch: true
  belongs_to :user, optional: true
end

# Publication - marks a board as publicly published
class Board::Publication < ApplicationRecord
  belongs_to :account, default: -> { board.account }
  belongs_to :board
  has_secure_token :key  # The public URL key
end
```

### Query Patterns

```ruby
# Finding open vs closed
Card.open                    # where.missing(:closure)
Card.closed                  # joins(:closure)

# Finding golden cards first
Card.with_golden_first       # left_outer_joins(:goldness).order(...)

# Finding active vs postponed
Card.active                  # open.published.where.missing(:not_now)
Card.postponed               # open.published.joins(:not_now)
```

---

## Views & Turbo/Hotwire Patterns

### Turbo Streams for Partial Updates

```erb
<%# app/views/cards/comments/create.turbo_stream.erb %>
<%= turbo_stream.before [@card, :new_comment],
    partial: "cards/comments/comment",
    locals: { comment: @comment } %>

<%= turbo_stream.update [@card, :new_comment],
    partial: "cards/comments/new",
    locals: { card: @card } %>
```

### Morphing for Complex Updates

```erb
<%# app/views/cards/update.turbo_stream.erb %>
<%= turbo_stream.replace dom_id(@card, :card_container),
    partial: "cards/container",
    method: :morph,
    locals: { card: @card.reload } %>
```

### Turbo Stream Subscriptions in Views

```erb
<%# app/views/cards/show.html.erb %>
<%= turbo_stream_from @card %>
<%= turbo_stream_from @card, :activity %>

<div data-controller="beacon" data-beacon-url-value="<%= card_reading_path(@card) %>">
  <%= render "cards/container", card: @card %>
  <%= render "cards/messages", card: @card %>
</div>
```

### Partials Over ViewComponents

```erb
<%# Use standard partials %>
<%= render "cards/container", card: @card %>
<%= render "cards/display/perma/meta", card: @card %>

<%# With caching %>
<% cache card do %>
  <section id="<%= dom_id(card, :card_container) %>">
    <%= render "cards/container/content", card: card %>
  </section>
<% end %>
```

### Stimulus Controllers are Focused

```javascript
// app/javascript/controllers/auto_submit_controller.js
// Single-purpose: auto-submit forms

// app/javascript/controllers/dialog_controller.js
// Single-purpose: manage dialogs

// app/javascript/controllers/beacon_controller.js
// Single-purpose: track views/reads
```

---

## Background Jobs

### Shallow Jobs, Rich Models

Jobs just call model methods:

```ruby
# app/jobs/notify_recipients_job.rb
class NotifyRecipientsJob < ApplicationJob
  def perform(notifiable)
    notifiable.notify_recipients
  end
end

# The model does the work
module Notifiable
  def notify_recipients
    Notifier.for(self)&.notify
  end
end
```

### `_later` and `_now` Convention

```ruby
module Card::Readable
  def mark_as_read_later(user:)
    MarkCardAsReadJob.perform_later(self, user)
  end

  def mark_as_read_now(user:)
    # Actual implementation
    readings.find_or_create_by!(user: user).touch
  end
end
```

### Database-Backed Jobs (Solid Queue)

No Redis. Jobs stored in the database:

```ruby
# config/recurring.yml
production:
  deliver_bundled_notifications:
    command: "Notification::Bundle.deliver_all_later"
    schedule: every 30 minutes

  auto_postpone_all_due:
    command: "Card.auto_postpone_all_due"
    schedule: every hour at minute 50

  cleanup_magic_links:
    command: "MagicLink.cleanup"
    schedule: every 4 hours
```

---

## Testing Approach

### Minitest, Not RSpec

```ruby
# test/models/card_test.rb
class CardTest < ActiveSupport::TestCase
  setup do
    Current.session = sessions(:david)
  end

  test "create assigns a number to the card" do
    card = Card.create!(title: "Test", board: boards(:writebook), creator: users(:david))
    assert_equal accounts("37s").reload.cards_count, card.number
  end

  test "closed" do
    assert_equal [cards(:shipping)], Card.closed
  end
end
```

### Integration Tests for Controllers

```ruby
# test/controllers/cards/closures_controller_test.rb
class Cards::ClosuresControllerTest < ActionDispatch::IntegrationTest
  setup do
    sign_in_as :kevin
  end

  test "create" do
    card = cards(:logo)

    assert_changes -> { card.reload.closed? }, from: false, to: true do
      post card_closure_path(card), as: :turbo_stream
    end
  end

  test "destroy as JSON" do
    card = cards(:shipping)
    assert card.closed?

    delete card_closure_path(card), as: :json

    assert_response :no_content
    assert_not card.reload.closed?
  end
end
```

### Fixtures Over Factories

```yaml
# test/fixtures/cards.yml
logo:
  account: 37s
  board: writebook
  creator: david
  title: "Logo Design"
  number: 1
  status: published

shipping:
  account: 37s
  board: writebook
  creator: david
  title: "Shipping"
  number: 2
  status: published
```

---

## What They Deliberately Avoid

### No Service Objects

```ruby
# BAD: Service object pattern
class CloseCardService
  def initialize(card, user)
    @card = card
    @user = user
  end

  def call
    @card.transaction do
      @card.create_closure!(user: @user)
      @card.track_event(:closed)
    end
  end
end

# GOOD: Method on model
class Card < ApplicationRecord
  def close(user: Current.user)
    transaction do
      create_closure!(user: user)
      track_event :closed, creator: user
    end
  end
end
```

### No Form Objects (Usually)

```ruby
# Exception: Signup is form-like but still an ActiveModel
class Signup
  include ActiveModel::Model
  include ActiveModel::Attributes
  include ActiveModel::Validations

  attr_accessor :full_name, :email_address, :identity

  validates :email_address, format: { with: URI::MailTo::EMAIL_REGEXP }, on: :identity_creation

  def create_identity
    @identity = Identity.find_or_create_by!(email_address: email_address)
    @identity.send_magic_link for: :sign_up
  end

  def complete
    if valid?(:completion)
      create_account
      true
    end
  end
end
```

### No Decorators/Presenters

View helpers and partials handle presentation:

```ruby
# app/helpers/cards_helper.rb
module CardsHelper
  def card_article_tag(card, **options, &block)
    classes = [
      options.delete(:class),
      ("golden-effect" if card.golden?),
      ("card--postponed" if card.postponed?)
    ].compact.join(" ")

    tag.article(class: classes, **options, &block)
  end
end
```

### No GraphQL

REST with Turbo is sufficient. API is JSON only when needed:

```ruby
def create
  @card.close

  respond_to do |format|
    format.turbo_stream { render_card_replacement }
    format.json { head :no_content }
  end
end
```

---

## Naming Conventions

### Verb Methods for Actions

```ruby
# GOOD: Action verbs
card.close
card.reopen
card.gild
card.ungild
card.postpone
card.resume
board.publish
board.unpublish

# BAD: Set-style methods
card.set_closed(true)
card.update_status(:closed)
```

### Predicate Methods for State

```ruby
# GOOD: Question methods
card.closed?
card.open?
card.golden?
card.postponed?
card.active?
card.entropic?

# Derived from presence of related record
def closed?
  closure.present?
end

def golden?
  goldness.present?
end
```

### Concern Naming

Concerns are adjectives describing capability:

- `Closeable` - can be closed
- `Publishable` - can be published
- `Watchable` - can be watched
- `Assignable` - can be assigned
- `Searchable` - can be searched
- `Eventable` - tracks events

### Controller Naming

Controllers are nouns matching the resource:

- `Cards::ClosuresController` - manages card closures
- `Cards::GoldnessesController` - manages card goldness
- `Boards::PublicationsController` - manages board publications

---

## Product Design Inferences

Based on the codebase, here are product decisions that emerge from technical constraints:

### 1. Entropy System (Auto-Postpone)

Cards automatically get "postponed" after inactivity. This is technically simple (a recurring job + a database record) but creates a product feature that keeps todo lists from becoming graveyards.

```ruby
# Simple to implement
class Card::Entropic
  class_methods do
    def auto_postpone_all_due
      due_to_be_postponed.find_each do |card|
        card.auto_postpone(user: card.account.system_user)
      end
    end
  end
end
```

### 2. State Records Enable Rich UI

Because states are records (not booleans), the UI can show:
- When something happened
- Who did it
- Filter by actor

```erb
<%= "Closed #{time_ago_in_words(card.closed_at)} by #{card.closed_by.name}" %>
```

### 3. Multi-Tenancy via URL

Account ID in URL (`/12345678/boards/...`) makes:
- Deep linking work naturally
- Sharing URLs include context
- No subdomain DNS complexity

### 4. Passwordless Auth

Magic links mean:
- No password reset flows
- No password storage liability
- Works across devices (email is the transfer mechanism)

### 5. Golden Cards (Not Priority Numbers)

Binary "golden" state is simpler than priority numbers. Forces decisions: is this important or not? No priority 1 vs 2 vs 3 debates.

### 6. Database-Backed Everything

No Redis means:
- One fewer infrastructure component
- Transactions work across jobs/cache/websockets
- Simpler backup/restore

---

## Summary: The 37signals Way

1. **Start with vanilla Rails** - Don't add abstractions until you feel the pain
2. **Models are rich** - Business logic lives in models, not services
3. **Controllers are thin** - Just orchestration and response formatting
4. **Everything is CRUD** - New resource over new action
5. **State is records** - Not boolean columns
6. **Concerns are compositions** - Horizontal behavior sharing
7. **Build before buying** - Auth, search, jobs - all custom
8. **Database is king** - No Redis, no Elasticsearch
9. **Test with fixtures** - Deterministic, fast, simple
10. **Ship incrementally** - Commit history shows many small changes

The best code is the code you don't write. The second best is the code that's obviously correct. The 37signals codebase optimizes for both.
