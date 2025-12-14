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
15. [HTTP Caching Patterns](#http-caching-patterns)
16. [Multi-Tenancy Deep Dive](#multi-tenancy-deep-dive)
17. [Database Patterns](#database-patterns)
18. [Stimulus Controller Patterns](#stimulus-controller-patterns)
19. [Event Tracking & Activity System](#event-tracking--activity-system)
20. [PORO Patterns](#poro-patterns-plain-old-ruby-objects)
21. [API Design Patterns](#api-design-patterns)
22. [Error Handling & Validation](#error-handling--validation)
23. [Configuration & Environment](#configuration--environment)
24. [Mailer Patterns](#mailer-patterns)
25. [Code Evolution Patterns](#code-evolution-patterns)

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

## HTTP Caching Patterns

37signals uses HTTP caching extensively via ETags and conditional GET requests.

### `fresh_when` for Conditional GET

Controllers declare cache dependencies with `fresh_when`:

```ruby
class Cards::AssignmentsController < ApplicationController
  def new
    @assigned_to = @card.assignees.active.alphabetically
    @users = @board.users.active.alphabetically.where.not(id: @card.assignees)
    fresh_when etag: [@users, @card.assignees]  # Returns 304 if unchanged
  end
end

class Cards::WatchesController < ApplicationController
  def show
    fresh_when etag: @card.watch_for(Current.user) || "none"
  end
end

class Boards::ColumnsController < ApplicationController
  def show
    set_page_and_extract_portion_from @column.cards.active.latest
    fresh_when etag: @page.records
  end
end
```

### Global ETags in ApplicationController

```ruby
class ApplicationController < ActionController::Base
  etag { "v1" }  # Bump to bust all caches on deploy
end
```

### Concern-Level ETags

```ruby
module CurrentTimezone
  included do
    etag { timezone_from_cookie }
  end
end

module Authentication
  included do
    etag { Current.identity.id if authenticated? }
  end
end
```

**Why timezone affects caching:**

Pages display times like "Created 2 hours ago" or "Dec 14, 3:00 PM". These are rendered server-side in the user's timezone (via `Time.use_zone`). If you cache the HTML and serve it to everyone:

- User in NYC sees "3:00 PM" ✓
- User in London gets same cached response, sees "3:00 PM" ✗ (should be "8:00 PM")

By including timezone in the ETag:
1. NYC user gets ETag `"abc123-America/New_York"`
2. London user's `If-None-Match` doesn't match → gets fresh response
3. Each timezone gets its own cached version

Same logic for `Current.identity.id` - personalized content ("You commented...") can't be shared across users.

### Complex ETags

```ruby
fresh_when etag: [@board, @page.records, @user_filtering]
fresh_when etag: [@filters, @boards, @tags, @users]
```

---

## Multi-Tenancy Deep Dive

URL-based multi-tenancy: `/{account_id}/boards/...`

### The Middleware Pattern

```ruby
# config/initializers/tenanting/account_slug.rb
module AccountSlug
  PATTERN = /(\d{7,})/  # 7+ digit account IDs
  PATH_INFO_MATCH = /\A(\/#{AccountSlug::PATTERN})/

  class Extractor
    def call(env)
      request = ActionDispatch::Request.new(env)

      if request.path_info =~ PATH_INFO_MATCH
        # Move /{account_id} from PATH_INFO to SCRIPT_NAME
        # Rails thinks it's "mounted" at this path
        request.engine_script_name = request.script_name = $1
        request.path_info = $'.empty? ? "/" : $'
        env["fizzy.external_account_id"] = $2.to_i
      end

      if env["fizzy.external_account_id"]
        account = Account.find_by(external_account_id: env["fizzy.external_account_id"])
        Current.with_account(account) { @app.call env }
      else
        Current.without_account { @app.call env }
      end
    end
  end
end
```

**Benefits:**
- No subdomain DNS complexity
- Deep links work naturally
- `Current.account` available everywhere
- URL helpers auto-include account prefix

---

## Database Patterns

### UUIDs Everywhere

```ruby
create_table "cards", id: :uuid do |t|
  t.uuid "account_id", null: false
  t.uuid "board_id", null: false
  t.uuid "creator_id", null: false
end
```

### Every Model Has `account_id`

```ruby
class Comment < ApplicationRecord
  belongs_to :account, default: -> { card.account }
end
```

### No Foreign Keys

Constraints removed for flexibility:

```ruby
class RemoveAllForeignKeyConstraints < ActiveRecord::Migration[8.2]
  def change
    # All foreign keys removed
  end
end
```

### Simple Migrations

```ruby
class AddPurposeToMagicLinks < ActiveRecord::Migration[8.2]
  def change
    add_column :magic_links, :purpose, :string, default: "sign_in"
  end
end
```

---

## Stimulus Controller Patterns

52 controllers split: **62% reusable, 38% domain-specific**.

### Reusable/Generic Controllers

```javascript
// toggle_class_controller.js
export default class extends Controller {
  static classes = ["toggle"]
  toggle() { this.element.classList.toggle(this.toggleClass) }
}

// copy_to_clipboard_controller.js
export default class extends Controller {
  static values = { content: String }
  async copy(e) {
    e.preventDefault()
    await navigator.clipboard.writeText(this.contentValue)
  }
}
```

### Domain-Specific Controllers

```javascript
// drag_and_drop_controller.js
export default class extends Controller {
  static targets = ["item", "container"]
  static classes = ["draggedItem", "hoverContainer"]

  async drop(event) {
    const container = this.#containerContaining(event.target)
    await this.#submitDropRequest(this.dragItem, container)
  }
}
```

### Design Rules

1. **Single responsibility** - One behavior per controller
2. **Configuration via values/classes** - `static values`, `static classes`
3. **Events for communication** - `this.dispatch("show")`
4. **Private methods with `#`** - `this.#handleSubmitEnd()`
5. **Small file size** - Most under 50 lines

---

## Event Tracking & Activity System

### The Event Model

Events are the single source of truth for activity:

```ruby
class Event < ApplicationRecord
  include Notifiable, Particulars, Promptable

  belongs_to :account, default: -> { board.account }
  belongs_to :board
  belongs_to :creator, class_name: "User"
  belongs_to :eventable, polymorphic: true  # Card, Comment, etc.

  after_create -> { eventable.event_was_created(self) }
  after_create_commit :dispatch_webhooks
end
```

### The Eventable Concern

```ruby
module Eventable
  extend ActiveSupport::Concern

  included do
    has_many :events, as: :eventable, dependent: :destroy
  end

  def track_event(action, creator: Current.user, board: self.board, **particulars)
    if should_track_event?
      board.events.create!(
        action: "#{eventable_prefix}_#{action}",
        creator:, board:, eventable: self,
        particulars:
      )
    end
  end
end
```

### Card-Specific Event Tracking

```ruby
module Card::Eventable
  include ::Eventable

  included do
    after_save :track_title_change, if: :saved_change_to_title?
  end

  def event_was_created(event)
    transaction do
      create_system_comment_for(event)
      touch_last_active_at unless was_just_published?
    end
  end

  private
    def track_title_change
      if title_before_last_save.present?
        track_event "title_changed", particulars: {
          old_title: title_before_last_save,
          new_title: title
        }
      end
    end
end
```

### Event Actions

```ruby
PERMITTED_ACTIONS = %w[
  card_assigned
  card_closed
  card_postponed
  card_auto_postponed
  card_board_changed
  card_published
  card_reopened
  card_sent_back_to_triage
  card_triaged
  card_unassigned
  comment_created
]
```

### Event Particulars (JSON metadata)

```ruby
module Event::Particulars
  included do
    store_accessor :particulars, :assignee_ids
  end

  def assignees
    @assignees ||= User.where(id: assignee_ids)
  end
end

# Usage
track_event "title_changed", particulars: {
  old_title: "Old",
  new_title: "New"
}
```

### Webhooks Driven by Events

```ruby
class Webhook < ApplicationRecord
  PERMITTED_ACTIONS = %w[card_assigned card_closed ...]

  has_many :deliveries, dependent: :delete_all
  serialize :subscribed_actions, type: Array, coder: JSON

  def for_slack?
    url.match? %r{//hooks\.slack\.com/services/}i
  end

  def for_campfire?
    url.match? %r{/rooms/\d+/\d+-[^\/]+/messages\Z}i
  end
end
```

---

## PORO Patterns (Plain Old Ruby Objects)

POROs are namespaced under their parent model:

### Nested Under Model Namespace

```
app/models/
├── event.rb
├── event/
│   ├── description.rb      # PORO
│   ├── particulars.rb      # Concern
│   └── promptable.rb       # Concern
├── card.rb
├── card/
│   ├── eventable/
│   │   └── system_commenter.rb  # PORO
│   ├── closeable.rb        # Concern
│   ├── golden.rb           # Concern
│   └── goldness.rb         # ActiveRecord
```

### PORO for Presentation Logic

```ruby
class Event::Description
  include ActionView::Helpers::TagHelper
  include ERB::Util

  attr_reader :event, :user

  def initialize(event, user)
    @event, @user = event, user
  end

  def to_html
    to_sentence(creator_tag, card_title_tag).html_safe
  end

  def to_plain_text
    to_sentence(creator_name, card.title)
  end

  private
    def action_sentence(creator, card_title)
      case event.action
      when "card_closed"
        %(#{creator} moved #{card_title} to "Done")
      when "card_reopened"
        "#{creator} reopened #{card_title}"
      # ...
      end
    end
end
```

### PORO for Business Logic

```ruby
class Card::Eventable::SystemCommenter
  include ERB::Util

  attr_reader :card, :event

  def initialize(card, event)
    @card, @event = card, event
  end

  def comment
    return unless comment_body.present?
    card.comments.create!(
      creator: card.account.system_user,
      body: comment_body,
      created_at: event.created_at
    )
  end

  private
    def comment_body
      case event.action
      when "card_closed"
        "<strong>Moved</strong> to "Done" by #{creator_name}"
      # ...
      end
    end
end
```

### PORO for View Context

```ruby
class User::Filtering
  attr_reader :user, :filter, :expanded

  delegate :as_params, :single_board, to: :filter

  def initialize(user, filter, expanded: false)
    @user, @filter, @expanded = user, filter, expanded
  end

  def boards
    @boards ||= user.boards.ordered_by_recently_accessed
  end

  def cache_key
    ActiveSupport::Cache.expand_cache_key(
      [user, filter, expanded?, boards, tags, users, filters],
      "user-filtering"
    )
  end
end
```

### When to Use POROs

1. **Presentation logic** - `Event::Description` formats events for display
2. **Complex operations** - `SystemCommenter` creates comments from events
3. **View context bundling** - `User::Filtering` collects filter UI state
4. **NOT service objects** - POROs are model-adjacent, not controller-adjacent

---

## API Design Patterns

### Same Controllers, Different Format

```ruby
def create
  @comment = @card.comments.create!(comment_params)

  respond_to do |format|
    format.turbo_stream
    format.json { head :created, location: card_comment_path(@card, @comment) }
  end
end
```

### Consistent Response Codes

| Action | Success Code |
|--------|--------------|
| Create | `201 Created` + `Location` header |
| Update | `204 No Content` |
| Delete | `204 No Content` |

### Bearer Token Authentication

```ruby
def authenticate_by_bearer_token
  if token = request.authorization&.match(/^Bearer (.+)$/)&.[](1)
    if access_token = AccessToken.find_by_token(token)
      set_current_session_from_access_token(access_token)
    end
  end
end
```

---

## Error Handling & Validation

### Minimal Validations

```ruby
class Account < ApplicationRecord
  validates :name, presence: true  # That's it
end

class Identity < ApplicationRecord
  validates :email_address, format: { with: URI::MailTo::EMAIL_REGEXP }
end
```

### Contextual Validations

```ruby
class Signup
  validates :email_address, format: { with: URI::MailTo::EMAIL_REGEXP }, on: :identity_creation
  validates :full_name, :identity, presence: true, on: :completion
end
```

### Let It Crash (Bang Methods)

```ruby
def create
  @comment = @card.comments.create!(comment_params)  # Raises on failure
end
```

---

## Configuration & Environment

### ENV Pattern: Fetch with Defaults

```ruby
# Use ENV.fetch with sensible defaults
ENV.fetch("PORT", 3000)
ENV.fetch("SMTP_PORT", "587").to_i
ENV.fetch("MULTI_TENANT", "true")
ENV.fetch("DATABASE_ADAPTER", saas? ? "mysql" : "sqlite")

# Only use ENV[] (no default) for optional values
ENV["MYSQL_PASSWORD"]
ENV["VAPID_PRIVATE_KEY"]
```

### Database Configuration: Multiple Databases

```yaml
# config/database.mysql.yml
default: &default
  adapter: trilogy
  host: <%= ENV.fetch("MYSQL_HOST", "127.0.0.1") %>
  port: <%= ENV.fetch("MYSQL_PORT", "3306") %>
  pool: 50
  timeout: 5000

production:
  primary:
    <<: *default
    database: fizzy_production
  cable:
    <<: *default
    database: fizzy_production_cable
  queue:
    <<: *default
    database: fizzy_production_queue
  cache:
    <<: *default
    database: fizzy_production_cache
```

Separate databases for:
- **Primary** - Main app data
- **Cable** - ActionCable/WebSocket messages
- **Queue** - Solid Queue jobs
- **Cache** - Solid Cache data

### SQLite for Simple Deployments

```yaml
# config/database.sqlite.yml
production:
  primary:
    adapter: sqlite3
    database: storage/production.sqlite3
    pool: 5
```

### Dynamic Database Selection

```ruby
# lib/fizzy.rb
module Fizzy
  def db_adapter
    @db_adapter ||= DbAdapter.new ENV.fetch("DATABASE_ADAPTER", saas? ? "mysql" : "sqlite")
  end
end

# config/database.yml
<%= ERB.new(File.read("config/database.#{Fizzy.db_adapter}.yml")).result %>
```

### Minimal Logging

Almost no explicit logging in app code:

```ruby
# Only 2 logging calls in entire app/models:
Rails.logger.error error
Rails.logger.error error.backtrace.join("\n")
```

Let Rails handle logging. Don't litter code with log statements.

### Configuration via Initializers

```ruby
# config/initializers/multi_tenant.rb
Rails.application.configure do
  config.after_initialize do
    Account.multi_tenant = ENV["MULTI_TENANT"] == "true" || config.x.multi_tenant.enabled == true
  end
end
```

### Important ENV Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `DATABASE_ADAPTER` | `sqlite`/`mysql` | Database type |
| `MULTI_TENANT` | `true` | Enable multi-tenancy |
| `SMTP_ADDRESS` | - | SMTP server |
| `MAILER_FROM_ADDRESS` | `support@fizzy.do` | From address |
| `ACTIVE_STORAGE_SERVICE` | `local` | Storage backend |
| `DISABLE_SSL` | `false` | Disable SSL in prod |
| `SOLID_QUEUE_IN_PUMA` | - | Run jobs in web process |

---

## Mailer Patterns

### Minimal Mailers

```ruby
class MagicLinkMailer < ApplicationMailer
  def sign_in_instructions(magic_link)
    @magic_link = magic_link
    @identity = @magic_link.identity
    mail to: @identity.email_address,
         subject: "Your Fizzy code is #{@magic_link.code}"
  end
end
```

### Bundled Notifications

```ruby
# config/recurring.yml
deliver_bundled_notifications:
  command: "Notification::Bundle.deliver_all_later"
  schedule: every 30 minutes
```

---

## Code Evolution Patterns

Analysis of the git history reveals how 37signals code evolves over time.

### Tests Ship With Features (Not TDD, Not Afterthought)

Tests are included in the same commit as features. Not strict TDD, but tests aren't added later either:

```
commit fa118a7 - "Add validation for the join code usage limit"
  app/models/account/join_code.rb                         |  3 +++
  app/controllers/account/join_codes_controller.rb        |  7 +++++--
  test/controllers/accounts/join_codes_controller_test.rb | 10 ++++++++++
```

The commit message focuses on the feature, but the test is there.

### Security Fixes Include Regression Tests

```ruby
# commit 83360ec - "Escape the names used to generate system comments"

# The fix:
def creator_name
  h event.creator.name  # ERB::Util.h for escaping
end

# The test (same commit):
test "escapes html in comment body" do
  users(:david).update! name: "<em>Injected</em>"
  @card.toggle_assignment users(:kevin)

  comment = @card.comments.last
  html = comment.body.body.to_html
  assert_includes html, "&lt;em&gt;Injected&lt;/em&gt;"  # Escaped!
  refute_includes html, "<em>Injected</em>"              # Not raw!
end
```

### Large Features Are Comprehensive

The storage tracking feature (commit 761d0b4) shipped with:
- 11 new model/concern files
- 2 jobs
- 2 migrations
- 7 test files with 1015+ lines of tests

All in one commit. Features ship complete.

### Refactoring Is Incremental

Logic gets moved to better locations over time:

```ruby
# Before: Controller concern checking environment
module SetTenant
  def single_tenant?
    ENV.fetch("SINGLE_TENANT", "false") == "true"
  end
end

# After: Model concern with class method
module MultiTenant
  class_methods do
    def accepting_signups?
      ENV.fetch("MULTI_TENANT", "true") == "true" || Account.none?
    end
  end
end
```

The pattern: **Start simple, extract when patterns emerge**.

### Consistency Refactors

When a pattern is established, older code gets updated to match:

```ruby
# commit 5cfe693 - "Change reaction admin permission check to be in-line with other controllers"

# Before: Inline permission check
def destroy
  @reaction = @comment.reactions.find(params[:id])
  if Current.user != @reaction.reacter
    head :forbidden
  else
    @reaction.destroy
  end
end

# After: Matches pattern from other controllers
before_action :set_reaction, only: [:destroy]
before_action :ensure_permision_to_administer_reaction, only: [:destroy]

def destroy
  @reaction.destroy
end

private
  def set_reaction
    @reaction = @comment.reactions.find(params[:id])
  end

  def ensure_permision_to_administer_reaction
    head :forbidden if Current.user != @reaction.reacter
  end
```

### Commit Message Patterns

Commit messages are concise and focus on the "what":

- `Add validation for the join code usage limit`
- `Escape the names used to generate system comments`
- `Wrap join code redemption in a lock`
- `Remove redundant include`
- `Refactor: use idiomatic .last instead of .order(:desc).first`

Not:
- ~~`[FIZZY-1234] Add validation for join code usage limit to prevent integer overflow errors when users enter extremely large numbers`~~

### Bug Fixes Are Small

Race condition fix (commit c8a5d01) touched 1 file, 2 lines:

```ruby
# Before
def redeem_if(&block)
  yield if redeemable?
end

# After
def redeem_if(&block)
  with_lock { yield if redeemable? }
end
```

### What This Tells Us

1. **Ship complete features** - Code + tests + migrations in one commit
2. **Refactor toward consistency** - When you establish a pattern, update old code
3. **Keep fixes small** - Bug fixes should be surgical
4. **Tests prove the fix** - Security/bug fixes include regression tests
5. **Start simple, extract later** - Don't over-engineer upfront

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
11. **Tests ship with features** - Not TDD, not afterthought, but together
12. **Refactor toward consistency** - Establish patterns, then update old code

The best code is the code you don't write. The second best is the code that's obviously correct. The 37signals codebase optimizes for both.
