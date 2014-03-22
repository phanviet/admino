# Admino

[![Gem Version](https://badge.fury.io/rb/admino.png)](http://badge.fury.io/rb/admino)
[![Build Status](https://travis-ci.org/cantierecreativo/admino.png?branch=v0.0.1)](https://travis-ci.org/cantierecreativo/admino)
[![Coverage Status](https://coveralls.io/repos/cantierecreativo/admino/badge.png?branch=master)](https://coveralls.io/r/cantierecreativo/admino?branch=master)
[![Code Climate](https://codeclimate.com/github/cantierecreativo/admino.png)](https://codeclimate.com/github/cantierecreativo/admino)

A minimal, object-oriented solution to generate Rails administrative index views. Through query objects and presenters, it features a customizable table generator and search forms with filtering/sorting.

## The philosophy behind it

The Rails ecosystem has many [full-fledged solutions to generate administrative interfaces](https://www.ruby-toolbox.com/categories/rails_admin_interfaces).

Although these tools are very handy to bootstrap a project quickly, they all obey the [80%-20% rule](http://en.wikipedia.org/wiki/Pareto_principle) and tend to be very invasive, often mixing up different concerns on a single responsibility level, thus making tests unbelievably difficult to setup and write.

A time comes when these all-encompassing tools get in the way. And that will be the moment where all the cumulated saved time will be wasted to solve a single, trivial problem with ugly workarounds and [epic facepalms](http://i.imgur.com/ghKDGyv.jpg).

So yes, if you're starting a small, short-lived project, go ahead with them, it will be fine! If you're building something that's more valuable or is meant to last longer, there are better alternatives.

### A modular approach to the problem

The great thing is that you don't need to write a lot of code to get a more maintainable and modular administrative area.
Gems like [Inherited Resources](https://github.com/josevalim/inherited_resources) and [Simple Form](https://github.com/plataformatec/simple_form), combined with [Rails 3.1+ template-inheritance](http://railscasts.com/episodes/269-template-inheritance) already give you ~90% of the time-saving features and the same super-DRY, declarative code that administrative interfaces offer, but with a far more relaxed contract.

If a particular controller or view needs something different from the standard CRUD/REST treatment, you can just avoid using those gems in that specific context, and fall back to standard Rails code. No workarounds, no facepalms. It seems easy, right? It is.

So what about Admino? Well, it complements the above-mentioned gems, giving you the the missing ~10%: a fast way to generate administrative index views.

## Installation

Add this line to your application's Gemfile:

    gem 'admino'

And then execute:

    $ bundle

## Admino::Query::Base

`Admino::Query::Base` implements the [Query object](http://martinfowler.com/eaaCatalog/queryObject.html) pattern, that is, an object responsible for returning a result set (ie. an `ActiveRecord::Relation`) based on business rules.

Given a `Task` model, we can generate a `TasksQuery` query object subclassing `Admino::Query::Base`:

```ruby
class TasksQuery < Admino::Query::Base
end
```

Each query object gets initialized with a hash of params, and features a `#scope` method that returns the filtered/sorted result set. As you may have guessed, query objects can be great companions to index controller actions:

```ruby
class TasksController < ApplicationController
  def index
    @query = TasksQuery.new(params)
    @tasks = @query.scope
  end
end
```

### Building the query itself

You can specify how a `TaskQuery` must build a result set through a simple DSL.

#### `starting_scope`

The `starting_scope` method is in charge of defining the scope that will start the filtering/ordering chain:

```ruby
class TasksQuery < Admino::Query::Base
  starting_scope { Task.all }
end

Task.create(title: 'Low priority task')

TaskQuery.new.scope.count # => 1
```

#### `search_field`

Once you define the following field:

```ruby
class TasksQuery < Admino::Query::Base
  # ...
  search_field :title_matches
end
```
The `#scope` method will check the presence of the `params[:query][:title_matches]` key. If it finds it, it will augment the query with a
named scope called `:title_matches`, expected to be found within the `Task` model, that needs to accept an argument.

```ruby
class Task < ActiveRecord::Base
  scope :title_matches, ->(text) {
    where('title ILIKE ?', "%#{text}%")
  }
end

Task.create(title: 'Low priority task')
Task.create(title: 'Fix me ASAP!!1!')

TaskQuery.new.scope.count # => 2
TaskQuery.new(query: { title_matches: 'ASAP' }).scope.count # => 1
```

#### `filter_by`

```ruby
class TasksQuery < Admino::Query::Base
  # ...
  filter_by :status, [:completed, :pending]
end
```

Just like a search field, with a declared filter group the `#scope` method will check the presence of a `params[:query][:status]` key. If it finds it (and its value corresponds to one of the declared scopes) it will augment the query the scope itself:

```ruby
class Task < ActiveRecord::Base
  scope :completed, -> { where(completed: true) }
  scope :pending,   -> { where(completed: false) }
end

Task.create(title: 'First task', completed: true)
Task.create(title: 'Second task', completed: true)
Task.create(title: 'Third task', completed: false)

TaskQuery.new.scope.count # => 3
TaskQuery.new(query: { status: 'completed' }).scope.count # => 2
TaskQuery.new(query: { status: 'pending' }).scope.count # => 1
TaskQuery.new(query: { status: 'foobar' }).scope.count # => 3
```

#### `sorting`

```ruby
class TasksQuery < Admino::Query::Base
  # ...
  sorting :by_due_date, :by_title
end
```

Once you declare some sorting scopes, the query object looks for a `params[:sorting]` key. If it exists (and corresponds to one of the declared scopes), it will augment the query with the scope itself. The model named scope will be called passing an argument that represents the direction of sorting (`:asc` or `:desc`).

The direction passed to the scope will depend on the value of `params[:sort_order]`, and will default to `:asc`:

```ruby
class Task < ActiveRecord::Base
  scope :by_due_date, ->(direction) { order(due_date: direction) }
  scope :by_title, ->(direction) { order(title: direction) }
end

expired_task = Task.create(due_date: 1.year.ago)
future_task = Task.create(due_date: 1.week.since)

TaskQuery.new(sorting: 'by_due_date', sort_order: 'desc').scope # => [ future_task, expired_task ]
TaskQuery.new(sorting: 'by_due_date', sort_order: 'asc').scope  # => [ expired_task, future_task ]
TaskQuery.new(sorting: 'by_due_date').scope                     # => [ expired_task, future_task ]
```

#### `ending_scope`

It's very common ie. to paginate a result set. The block declared in the `ending_scope` block will be always appended to the end of the chain:

```ruby
class TasksQuery < Admino::Query::Base
  ending_scope { |q| page(q.params[:page]) }
end
```

### Inspecting the query state

A query object supports various methods to inspect the available search fields, filters and sortings, and their state:

```ruby
query = TaskQuery.new
query.search_fields  # => [ #<Admino::Query::SearchField>, ... ]
query.filter_groups  # => [ #<Admino::Query::FilterGroup>, ... ]

search_field = query.search_field_by_name(:title_matches)

search_field.name      # => :title_matches
search_field.present?  # => true
search_field.value     # => 'ASAP'

filter_group = query.filter_group_by_name(:status)

filter_group.name                        # => :status
filter_group.scopes                      # => [ :completed, :pending ]
filter_group.active_scope                # => :completed
filter_group.is_scope_active?(:pending)  # => false

sorting = query.sorting                  # => #<Admino::Query::Sorting>
sorting.scopes                           # => [ :by_title, :by_due_date ]
sorting.active_scope                     # => :by_due_date
sorting.is_scope_active?(:by_title)      # => false
sorting.ascending?                       # => true
```

### Presenting search form and filters to the user

Admino also offers a [Showcase presenter](https://github.com/stefanoverna/showcase) that makes it really easy to generate search forms and filtering links:

```erb
<%# instanciate the the query object presenter %>
<% query = present(@query) %>

<%# generate the search form %>
<%= query.form do |q| %>
  <p>
    <%= q.label :title_matches %>
    <%= q.text_field :title_matches %>
  </p>
  <p>
    <%= q.submit %>
  </p>
<% end %>

<%# generate the filtering links %>
<% query.filter_groups.each do |filter_group| %>
  <h6><%= filter_group.name %></h6>
  <ul>
    <% filter_group.scopes.each do |scope| %>
      <li>
        <%= filter_group.scope_link(scope) %>
      <li>
    <% end %>
  </ul>
<% end %>

<%# generate the sorting links %>
<h6>Sort by</h6>
<ul>
  <% query.sorting.scopes.each do |scope| %>
    <li>
      <%= query.sorting.scope_link(scope) %>
    </li>
  <% end %>
</ul>
```

The great thing is that:

* the search form gets automatically filled in with the last input the user submitted
* a `is-active` CSS class gets added to the currently active filter scopes
* if a particular filter link has been clicked and is now active, it is possible to deactivate it by clicking on the link again
* a `is-asc`/`is-desc` CSS class gets added to the currently active sorting scope
* if a particular sorting scope link has been clicked and is now in ascending order, it is possible to make it descending by clicking on the link again

### Simple Form support

The presenter also offers a `#simple_form` method to make it work with [Simple Form](https://github.com/plataformatec/simple_form) out of the box.

### I18n

To localize the search form labels, as well as the group filter names and scope links, please refer to the following YAML file:

```yaml
en:
  query:
    attributes:
      tasks_query:
        title_matches: 'Title contains'
    filter_groups:
      tasks_query:
        status:
          name: 'Filter by status'
          scopes:
            completed: 'Completed'
            pending: 'Pending'
    sorting_scopes:
      task_query:
        by_due_date: 'By due date'
        by_title: 'By title'
```

### Output customisation

The presenter supports a number of optional arguments that allow a great amount of flexibility regarding customisation of CSS classes, labels and HTML attributes. Please refer to the tests for the details.

### Overwriting the starting scope

Suppose you have to filter the tasks based on the `@current_user` work group. You can easily provide an alternative starting scope from the controller passing it as an argument to the `#scope` method:

```ruby
def index
  @query = TasksQuery.new(params)
  @project_tasks = @query.scope(@current_user.team.tasks)
end
```

### Coertions

Admino can perform automatic coertions from a param string input to the type needed by the model named scope:

```ruby
class TasksQuery < Admino::Query::Base
  # ...
  field :due_date_from, coerce: :to_date
  field :due_date_to, coerce: :to_date
end
```
The following coertions are available:

* `:to_boolean`
* `:to_constant`
* `:to_date`
* `:to_datetime`
* `:to_decimal`
* `:to_float`
* `:to_integer`
* `:to_symbol`
* `:to_time`

If a specific coercion cannot be performed with the provided input, the scope won't be chained.

Please see the [`Coercible::Coercer::String`](https://github.com/solnic/coercible/blob/master/lib/coercible/coercer/string.rb) class for details.

### Default sorting

If you need to setup a default sorting, you can pass some optional arguments to a `scoping` declaration:

```ruby
class TasksQuery < Admino::Query::Base
  # ...
  sorting :by_due_date, :by_title,
          default_scope: :by_due_date,
          default_direction: :desc
end
```

## Admino::Table::Presenter

Admino offers a [Showcase collection presenter](https://github.com/stefanoverna/showcase) that makes it really easy to generate HTML tables from a set of records:

```erb
<% tasks = present_collection(@tasks) %>

<%= Admino::Table::Presenter.new(@tasks, Task, self).to_html do |row, record| %>
  <%= row.column :title %>
  <%= row.column :completed %>
  <%= row.column :due_date %>
<% end %>
```

```html
<table>
  <thead>
    <tr>
      <th role='title'>
        Title
      </th>
      <th role='completed'>
        Completed
      </th>
      <th role='due_date'>
        Due date
      </th>
    </tr>
  <thead>
  <tbody>
    <tr id='task_1' class='is-even'>
      <td role='title'>
        Call mum ASAP
      </td>
      <td role='completed'>
        false
      </td>
      <td role='due_date'>
        2013-02-04
      </td>
    </tr>
    <tr id='task_2' class='is-odd'>
      <!-- ... -->
    </tr>
  <tbody>
</table>
```

### Actions

WIP

### Customising the output

WIP

### Subclassing

```ruby
class TableCollectionPresenter < Admino::Table::Presenter
end
```

### I18n

WIP

