# Migrating from Rails to Sequel ORM + Plain Ruby

A practical guide based on stripping Rails 7.1 from the Peregrine Penetrator Scanner — a CLI tool that was using Rails as an ORM wrapper but never used controllers, jobs, mailers, views, or an HTTP server. The migration replaced ActiveRecord with Sequel ORM and the Rails boot sequence with a plain Ruby module.

## Why We Migrated

The scanner runs as an ephemeral CLI tool on GCP VMs. Rails loaded ~50,000 lines of framework code, ~300MB RAM, 3-5s boot time, and carried ~38 gem dependencies. After migration: ~80MB RAM, <1s boot, ~20 gems. Attack surface dropped significantly (Rails has had ~20 critical CVEs since 2020).

## The Migration in Numbers

| Metric | Before (Rails) | After (Sequel) |
|--------|---------------|-----------------|
| Gems | 38 | 20 |
| Boot time | 3-5s | <1s |
| Memory | ~300MB | ~80MB |
| Framework code loaded | ~50,000 lines | ~2,000 lines |
| Test examples | 536 | 536 (531 passing, 5 edge cases) |

## Migration Strategy

We did this in a single feature branch with incremental commits, each leaving the app in a testable state:

1. Create `lib/penetrator.rb` boot module alongside existing Rails boot
2. Write Sequel migrations alongside existing AR migrations
3. Write Sequel models in `lib/models/` alongside existing AR models in `app/models/`
4. Create `spec/sequel_helper.rb` alongside `spec/rails_helper.rb`
5. Bulk replace `Rails.root` → `Penetrator.root`, `Rails.logger` → `Penetrator.logger`
6. Port service queries from AR DSL to Sequel DSL
7. Create `bin/scan` CLI to replace `rake scan:run`
8. Delete all Rails boilerplate, switch all specs to `sequel_helper`
9. Fix remaining issues and verify

Steps 1-4 were additive (new files only), so existing Rails tests kept passing throughout. The big switchover happened at step 8.

## Gotcha #1: Sequel Serialization Doesn't Detect In-Place Mutations

**The single most impactful issue.** Sequel's `plugin :serialization, :json` tracks changes by object identity. If you mutate a hash in place and call `.update()`, the change is silently dropped.

```ruby
# BROKEN — change is silently lost
statuses = scan.tool_statuses || {}
statuses['zap'] = { status: 'completed' }
scan.update(tool_statuses: statuses)  # Same object reference — Sequel thinks nothing changed

# WORKS — assign a new object
scan.tool_statuses = (scan.tool_statuses || {}).merge('zap' => { status: 'completed' })
scan.save_changes
```

**Fix**: Always use the setter (`model.col = new_value`) and `save_changes` for serialized columns. Never mutate the existing hash/array in place.

## Gotcha #2: `scan.findings` Returns an Array, Not a Dataset

In ActiveRecord, `scan.findings` returns a CollectionProxy that supports `.where`, `.order`, `.non_duplicate`, etc. In Sequel, `scan.findings` eagerly loads and returns a plain Ruby Array.

```ruby
# BROKEN — Array doesn't have .non_duplicate or .where
scan.findings.non_duplicate.by_severity
scan.findings.where(severity: 'high')

# WORKS — use _dataset to get a Sequel::Dataset
scan.findings_dataset.non_duplicate.by_severity
scan.findings_dataset.where(severity: 'high')
```

**This was the most common breakage pattern** — every service file that queried through an association needed `_dataset`. We found ~15 occurrences across audit_logger, scan_results_exporter, big_query_logger, executive_summarizer, ticketing_service, scan_callback_service, email_notifier, finding_normalizer, report_generator, and ai_analyzer.

## Gotcha #3: No Rails Autoloading Means Load Order Matters

Rails autoloads constants lazily. Without it, if `HtmlReport` includes `MarkdownConverter`, the module must be loaded first. If `DawnScanner` extends `ScannerBase`, the base class must load first.

```ruby
# Our load order in lib/penetrator.rb:
# 1. Report generator modules (dependency order: helpers → methodology → styles → converters → reports)
# 2. Base classes (scanner_base.rb)
# 3. Subdirectory files (scanners/, parsers/, ai/, cve_clients/)
# 4. Top-level service files (scan_orchestrator.rb, report_generator.rb, etc.)
```

**Fix**: Explicit load ordering in the boot module. We maintained a hardcoded list for report generators (which have complex include chains) and used a depth-first directory scan for everything else.

## Gotcha #4: Gems Need Explicit `require`

Rails auto-requires gems listed in the Gemfile. Without Rails, `Anthropic::Client`, `Mail`, `Faraday` etc. are undefined.

```ruby
# In lib/penetrator.rb boot module:
require 'faraday'
require 'anthropic'
require 'mail'
```

**Symptoms**: `NameError: uninitialized constant Anthropic` in specs that mock `Anthropic::Client`.

## Gotcha #5: ActiveSupport Standalone Requires Full Init

Cherry-picking individual ActiveSupport extensions fails:

```ruby
# BROKEN — "undefined method `deprecator' for ActiveSupport:Module"
require 'active_support/core_ext/numeric/time'

# WORKS — full initialization
require 'active_support'
require 'active_support/core_ext'
```

Also: `ActiveSupport::Testing::TimeHelpers` (`freeze_time`, `travel_to`) is **not available** outside Rails. Replace with time range comparisons in specs.

## Gotcha #6: FactoryBot Needs Sequel Configuration

FactoryBot uses `save!` by default (AR method). Sequel uses `save`.

```ruby
# In spec/sequel_helper.rb:
FactoryBot.define do
  to_create { |instance| instance.save }
end
```

Also: Sequel requires associated objects to be persisted before assignment (they need a primary key). Use `strategy: :create` on association factories:

```ruby
factory :scan do
  association :target, strategy: :create  # Not just `target`
end
```

## Gotcha #7: Factory JSON Column Values

AR's `serialize :col, coder: JSON` accepts both strings and hashes. Sequel's serialization plugin auto-encodes hashes to JSON, so passing `.to_json` in factories double-encodes:

```ruby
# BROKEN — stored as "\"{\\\"description\\\":\\\"...\\\"}\"" (double-encoded)
evidence { { 'description' => 'test' }.to_json }

# WORKS — Sequel serializes automatically
evidence { { 'description' => 'test' } }
```

## Gotcha #8: Sequel Query DSL Differences

| ActiveRecord | Sequel | Notes |
|---|---|---|
| `.update!(attrs)` | `.update(attrs)` | No bang variant |
| `.create!` on association | `Model.create(parent_id: id, **attrs)` | No association proxy `.create!` |
| `.where.not(col: val)` | `.exclude(col: val)` | |
| `.find_or_create_by!(attrs)` | `.find_or_create(attrs)` | No bang |
| `.find_each` | `.paged_each` or `.each` | |
| `.group(:col).count` | `.group_and_count(:col).all.to_h { \|r\| [r[:severity], r[:count]] }` | Returns dataset, not hash |
| `.reload` | `.refresh` | |
| `Arel.sql(...)` | `Sequel.lit(...)` | |
| `ActiveRecord::RecordInvalid` | `Sequel::ValidationFailed` | |
| `validates :col, inclusion: { in: [...] }` | `validates_includes [...], :col` | Plugin: `validation_helpers` |
| `validates :col, uniqueness: { scope: :other }` | `validates_unique(:col) { \|ds\| ds.where(other: other_val) }` | Block for scoping |
| `scope :name, -> { where(...) }` | `dataset_module { def name; where(...); end }` | |
| `has_many :items` | `one_to_many :items` | |
| `belongs_to :parent` | `many_to_one :parent` | |
| `serialize :col, coder: JSON` | `plugin :serialization, :json, :col` | |

## Gotcha #9: `validates_unique` Only Checks on Save

AR's `validates :col, uniqueness: ...` is checked by `.valid?`. Sequel's `validates_unique` only validates when the record is actually saved (it runs a SQL query).

```ruby
# AR test pattern (works):
obj = build(:finding, fingerprint: 'abc')
expect(obj.valid?).to be false

# Sequel test pattern (validates_unique requires save):
expect { create(:finding, fingerprint: 'abc') }.to raise_error(Sequel::ValidationFailed)
```

## Gotcha #10: Validation Error Message Format

AR: `"Format is not included in the list"`. Sequel: `"format is not in range or set: [\"json\", ...]"`. If specs match on error messages, use loose regex:

```ruby
# Fragile:
expect { ... }.to raise_error(Sequel::ValidationFailed, /Format is not included/)

# Robust:
expect { ... }.to raise_error(Sequel::ValidationFailed, /format/i)
```

## What Stayed the Same

- **ActiveSupport core extensions** — `.present?`, `.blank?`, `.titleize`, `Time.current`, `1.hour.ago`, `7.days.from_now` all work standalone. This avoided touching 50+ files.
- **FactoryBot** — works identically with Sequel (with the `to_create` configuration above).
- **WebMock** — no changes needed.
- **SimpleCov** — no changes needed (remove `'rails'` profile: `SimpleCov.start` not `SimpleCov.start 'rails'`).
- **RSpec** — `rspec` gem replaces `rspec-rails`. All matchers work the same.
- **Service layer** — 51 service files needed only mechanical query DSL changes, no logic changes.

## Recommendation

If your Rails app doesn't use controllers, views, or an HTTP server — Sequel is a strong replacement. The migration is mechanical (mostly find-replace), the gotchas are well-documented above, and the result is dramatically lighter. Keep ActiveSupport as a standalone gem to avoid touching every file that uses `.present?`.
