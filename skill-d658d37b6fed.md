<!-- dchuk/source_monitor:.claude/skills/tdd-cycle/SKILL.md | https://github.com/dchuk/source_monitor/blob/HEAD/.claude/skills/tdd-cycle/SKILL.md -->

---
name: tdd-cycle
description: Guides Test-Driven Development workflow with Red-Green-Refactor cycle using Minitest and fixtures. Use when the user wants to implement a feature using TDD, write tests first, follow test-driven practices, or mentions red-green-refactor.
allowed-tools: Read, Write, Edit, Bash
---

# TDD Cycle — Minitest + Fixtures

## Project Conventions
- **Testing:** Minitest + fixtures (NEVER RSpec or FactoryBot)
- **Components:** ViewComponents for reusable UI (partials OK for simple one-offs)
- **Authorization:** Pundit policies (deny by default)
- **Jobs:** Solid Queue, shallow jobs, `_later`/`_now` naming
- **Frontend:** Hotwire (Turbo + Stimulus) + Tailwind CSS
- **State:** State-as-records for business state (booleans only for technical flags)
- **Architecture:** Rich models first, service objects for multi-model orchestration
- **Routing:** Everything-is-CRUD (new resource over new action)
- **Quality:** RuboCop (omakase) + Brakeman

## The Cycle

```
1. RED    → Write a failing test that describes desired behavior
2. GREEN  → Write the minimum code to pass the test
3. REFACTOR → Improve code while keeping tests green
4. REPEAT → Next behavior
```

## Workflow Checklist

```
TDD Progress:
- [ ] Step 1: Understand the requirement
- [ ] Step 2: Choose test type (model/controller/system/component)
- [ ] Step 3: Write failing test (RED)
- [ ] Step 4: Verify test fails correctly
- [ ] Step 5: Implement minimal code (GREEN)
- [ ] Step 6: Verify test passes
- [ ] Step 7: Refactor if needed
- [ ] Step 8: Verify tests still pass
```

## Step 1: Requirement Analysis

Before writing any code, understand:
- What is the expected input?
- What is the expected output/behavior?
- What are the edge cases?
- What errors should be handled?

## Step 2: Choose Test Type

| Test Type | Use For | Location |
|-----------|---------|----------|
| Model test | Validations, scopes, instance methods | `test/models/` |
| Controller test | HTTP flow, authorization, responses | `test/controllers/` |
| System test | Full user flows with JavaScript | `test/system/` |
| Service test | Business logic, complex operations | `test/services/` |
| Query test | Complex queries, correctness | `test/queries/` |
| Component test | ViewComponent rendering | `test/components/` |
| Policy test | Pundit authorization rules | `test/policies/` |
| Job test | Background job behavior | `test/jobs/` |
| Mailer test | Email content, recipients | `test/mailers/` |

## Step 3: Write Failing Test (RED)

### Model Test Template

```ruby
# test/models/post_test.rb
require "test_helper"

class PostTest < ActiveSupport::TestCase
  setup do
    @post = posts(:published)
  end

  test "requires title" do
    @post.title = nil
    assert_not @post.valid?
    assert_includes @post.errors[:title], "can't be blank"
  end

  test ".recent returns posts in descending order" do
    recent = posts(:recent)
    old = posts(:old)
    assert_equal [recent, old], Post.recent.to_a
  end

  test "#publish! creates a publication record" do
    post = posts(:draft)
    assert_difference "Publication.count", 1 do
      post.publish!(user: users(:admin))
    end
  end
end
```

### Controller (Integration) Test Template

```ruby
# test/controllers/posts_controller_test.rb
require "test_helper"

class PostsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:one)
    @post = posts(:one)
    sign_in_as @user
  end

  test "should get index" do
    get posts_url
    assert_response :success
  end

  test "should create post" do
    assert_difference("Post.count") do
      post posts_url, params: { post: { title: "New Post", body: "Content" } }
    end
    assert_redirected_to post_url(Post.last)
  end

  test "should not create post with invalid params" do
    assert_no_difference("Post.count") do
      post posts_url, params: { post: { title: "", body: "" } }
    end
    assert_response :unprocessable_entity
  end

  test "requires authentication" do
    sign_out
    get posts_url
    assert_redirected_to new_session_url
  end
end
```

### Service Test Template

```ruby
# test/services/orders/create_service_test.rb
require "test_helper"

class Orders::CreateServiceTest < ActiveSupport::TestCase
  setup do
    @user = users(:one)
    @product = products(:widget)
    @service = Orders::CreateService.new
  end

  test "creates order with valid params" do
    result = @service.call(user: @user, items: [{ product_id: @product.id, quantity: 2 }])

    assert result.success?
    assert_kind_of Order, result.data
    assert_equal @user, result.data.user
  end

  test "returns failure with empty items" do
    result = @service.call(user: @user, items: [])

    assert result.failure?
    assert_equal :empty_cart, result.code
  end

  test "wraps in transaction" do
    assert_no_difference "Order.count" do
      @service.call(user: @user, items: [{ product_id: 0, quantity: 1 }])
    end
  end
end
```

### System Test Template

```ruby
# test/system/posts_test.rb
require "application_system_test_case"

class PostsTest < ApplicationSystemTestCase
  setup do
    @user = users(:one)
    sign_in_as @user
  end

  test "creating a post" do
    visit new_post_url

    fill_in "Title", with: "My Post"
    fill_in "Body", with: "Post content here"
    click_button "Create Post"

    assert_text "Post created successfully"
    assert_text "My Post"
  end

  test "shows validation errors" do
    visit new_post_url
    click_button "Create Post"

    assert_text "can't be blank"
  end
end
```

### ViewComponent Test Template

```ruby
# test/components/status_badge_component_test.rb
require "test_helper"

class StatusBadgeComponentTest < ViewComponent::TestCase
  test "renders published badge" do
    render_inline(StatusBadgeComponent.new(status: :published))

    assert_selector ".badge", text: "Published"
    assert_selector ".bg-green-100"
  end

  test "renders draft badge" do
    render_inline(StatusBadgeComponent.new(status: :draft))

    assert_selector ".badge", text: "Draft"
    assert_selector ".bg-gray-100"
  end
end
```

### Policy Test Template

```ruby
# test/policies/post_policy_test.rb
require "test_helper"

class PostPolicyTest < ActiveSupport::TestCase
  setup do
    @owner = users(:one)
    @other = users(:two)
    @post = posts(:one) # belongs to @owner
  end

  test "owner can update" do
    assert PostPolicy.new(@owner, @post).update?
  end

  test "non-owner cannot update" do
    assert_not PostPolicy.new(@other, @post).update?
  end

  test "scope returns only authorized records" do
    scope = PostPolicy::Scope.new(@owner, Post).resolve
    assert_includes scope, @post
  end
end
```

## Step 4: Verify Failure

Run the test:
```bash
bin/rails test test/models/post_test.rb --verbose
```

The test MUST fail with a clear error. If it passes immediately, either:
- The behavior already exists
- The test isn't testing what you think

## Step 5: Implement (GREEN)

Write the MINIMUM code to pass:
- No optimization
- No edge case handling beyond what's tested
- No refactoring
- Just make it work

## Step 6: Verify Pass

```bash
bin/rails test test/models/post_test.rb --verbose
```

## Step 7: Refactor

Improve code while tests stay green:
- Extract methods for clarity
- Improve naming
- Remove duplication
- Simplify logic

**Rule:** Make ONE change at a time, run tests after EACH change.

## Step 8: Final Verification

Run all related tests:
```bash
bin/rails test
```

## Fixtures Best Practices

```yaml
# test/fixtures/posts.yml
published:
  title: Published Post
  body: This is published content
  user: one
  created_at: <%= 1.day.ago %>

draft:
  title: Draft Post
  body: This is draft content
  user: one

recent:
  title: Recent Post
  body: Recent content
  user: one
  created_at: <%= 1.hour.ago %>

old:
  title: Old Post
  body: Old content
  user: two
  created_at: <%= 1.year.ago %>
```

**Fixture naming tips:**
- Use descriptive names: `published`, `draft`, `admin_post`
- Reference other fixtures by name: `user: one`
- Use ERB for dynamic values: `<%= Time.current %>`

## Test Helper Patterns

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  # Use fixtures for all tests
  fixtures :all

  # Authentication helper
  def sign_in_as(user)
    post session_url, params: { email: user.email, password: "password" }
  end

  def sign_out
    delete session_url
  end
end
```

## Anti-Patterns to Avoid

1. **Testing implementation, not behavior** — Test what it does, not how
2. **Too many assertions per test** — One concept per test
3. **Brittle tests** — Don't assert exact timestamps or error messages
4. **Slow tests** — Prefer model tests over system tests when possible
5. **Skipping the RED step** — Always see it fail first
6. **Over-mocking** — Use real objects with fixtures when possible
