# Dossier

Dossier is a Rails engine that turns SQL into reports. Reports can be easily rendered in various formats, like HTML, CSV, XLS, and JSON.

- If you **hate** SQL, you can use whatever tool you like to generate it; for example, ActiveRecord's `to_sql`.
- If you **love** SQL, you can use every feature your database supports.

[![Gem Version](https://badge.fury.io/rb/dossier.svg)](http://badge.fury.io/rb/dossier)
[![Code Climate](https://codeclimate.com/github/adamhunter/dossier/badges/gpa.svg)](https://codeclimate.com/github/adamhunter/dossier)
[![Build Status](https://travis-ci.org/adamhunter/dossier.svg?branch=master)](https://travis-ci.org/adamhunter/dossier)
[![Coverage Status](https://coveralls.io/repos/adamhunter/dossier/badge.svg?branch=master&service=github)](https://coveralls.io/github/adamhunter/dossier?branch=master)
[![Dependency Status](https://gemnasium.com/adamhunter/dossier.svg)](https://gemnasium.com/adamhunter/dossier)

## Setup

Install the Dossier gem and create `config/dossier.yml`. This has the same format as Rails' `database.yml`, and can actually just be a symlink (from your `Rails.root`: `ln -s database.yml config/dossier.yml`).

## Routing

Dossier will add a route to your app so that `reports/fancy_ketchup` will instantiate and run a `FancyKetchupReport`. It will respond with whatever format was requested; for example `reports/fancy_ketchup.csv` will render the results as CSV.

## Formats

Dossier currently supports outputting to the following formats:

- HTML
- CSV
- XLS
- JSON

Any of these formats can be requested by using the appropriate format extension on the end of the report's URL.

## Basic Reports

In your app, create report classes under `app/reports`, with `Report` as the end of the class name. Define a `sql` method that returns the sql string to be sent to the database.

For example:

```ruby
# app/reports/fancy_ketchup_report.rb
class FancyKetchupReport < Dossier::Report
  def sql
    'SELECT * FROM ketchups WHERE fancy = true'
  end

  # Or, if you're using ActiveRecord and hate writing SQL:
  def sql
    Ketchup.where(fancy: true).to_sql
  end

end
```

If you need dynamic values that may be influenced by the user, **[do not interpolate them directly](http://xkcd.com/327/)**. Dossier provides a safer way to add them: any lowercase symbols in the query will be replaced by calling methods of the same name in the report. Return values will be **escaped by the database connection**.  Arrays will have all of their contents escaped, joined with a "," and wrapped in parentheses.

```ruby
# app/reports/fancy_ketchup_report.rb
class FancyKetchupReport < Dossier::Report
  def sql
    "SELECT * FROM ketchups WHERE price <= :max_price and brand IN :brands"
    # => "SELECT * FROM ketchups WHERE price <= 7 and brand IN ('Acme', 'Generic', 'SoylentRed')"
  end

  def max_price
    7
  end

  def brands
    %w[Acme Generic SoylentRed]
  end
end
```

## Header Formatting

By default, headers are generated by calling `titleize` on the column name from the result set.  To override this, define a `format_header` method in your report that returns what you want.  For example:

```ruby
class ProductMarginReport < Dossier::Report
  # ...
  def format_header(column_name)
    custom_headers = {
      margin_percentage: 'Margin %',
      absolute_margin:   'Margin $'
    }
    custom_headers.fetch(column_name.to_sym) { super }
  end
end
```

## Column Formatting

You can format any values in your results by defining a `format_` method for that column on your report class. For instance, to reverse the names of your employees:

```ruby
class EmployeeReport < Dossier::Report
  # ...
  def format_name(value)
    value.reverse
  end
end
```

Dossier also provides a `formatter` with access to all the standard Rails formatters. So to format all values in the `payment` column as currency, you could do:

```ruby
class MoneyLaunderingReport < Dossier::Report
  #...
  def format_payment(value)
    formatter.number_to_currency(value)
  end
end
```

In addition, the formatter provides Rails' URL helpers for use in your reports. For example, in a report of your least profitable accounts, you might want to add a link to change the salesperson assigned to that account.

```ruby
class LeastProfitableAccountsReport < Dossier::Report
  #...
  def format_account_id(value)
    formatter.url_formatter.link_to value, formatter.url_formatter.url_helpers.edit_accounts_path(value)
  end
end
```

The built-in `ReportsController` uses this formatting when rendering the HTML and JSON representations, but not when rendering the CSV or XLS.

If your formatting method takes a second argment, it will be given a hash of the values in the row.

```ruby
class MoneyLaunderingReport < Dossier::Report
  #...
  def format_payment(value, row)
    return "$0.00" if row[:recipient] == 'Jimmy The Squid'
    formatter.number_to_currency(value)
  end
end
```

## Hidden Columns

You may override `display_column?` in your report class in order to hide columns from the formatted results. For instance, you might select an employee's ID and name in order to generate a link from their name to their profile page, without actually displaying the ID value itself:

```ruby
class EmployeeReport < Dossier::Report
  # ...

  def display_column?(name)
    name != 'id'
  end

  def format_name(value, row)
    url = formatter.url_formatter.url_helpers.employee_path(row['id'])
    formatter.url_formatter.link_to(value, url)
  end
end
```

By default, all selected columns are displayed.

## Report Options and Footers

You may want to specify parameters for a report: which columns to show, a range of dates, etc. Dossier supports this via URL parameters, anything in `params[:options]` will be passed into your report's `initialize` method and made available via the `options` reader.

You can pass these options by hardcoding them into a link, or you can allow users to customize a report with a form. For example:

```ruby
# app/views/dossier/reports/employee.html.haml

= form_for report, as: :options, url: url_for, html: {method: :get} do |f|
  = f.label "Salary greater than:"
  = f.text_field :salary_greater_than
  = f.label "In Division:"
  = f.select_tag :in_division, divisions_collection
  = f.button "Submit"

= render template: 'dossier/reports/show', locals: {report: report}
```

It's up to you to use these options in generating your SQL query.

However, Dossier does support one URL parameter natively: if you supply a `footer` parameter with an integer value, the last N rows will be accesible via `report.results.footers` instead of `report.results.body`. The built-in `show` view renders those rows inside an HTML footer. This is an easy way to display a totals row or something similar.

## Styling

The default report views use a `<table class="dossier report">` for easy CSS styling.

## Additional View Customization

To further customize your results view, run the generator provided. The default will provide 'app/views/dossier/reports/show'.

```ruby
rails generate dossier:views
```
You may pass a filename as an argument. This example creates 'app/views/dossier/reports/account_tracker.html.haml'.

```ruby
rails generate dossier:views account_tracker
```

## Callbacks

To produce report results, Dossier builds your query and executes it in separate steps. It uses [ActiveSupport::Callbacks](http://api.rubyonrails.org/classes/ActiveSupport/Callbacks.html) to define callbacks for `build_query` and `execute`. Therefore, you may provide callbacks similar to these:

```ruby
set_callback :build_query, :before, :run_my_stored_procedure
set_callback :execute,     :after do
  mangle_results
end
```

## Using Reports Outside of Dossier::ReportsController

### With Other Controllers

You can use Dossier reports in your own controllers and views. For example, if you wanted to render two reports on a page with other information, you might do this in a controller:

```ruby
class ProjectsController < ApplicationController

  def show
    @project                = Project.find(params[:id])
    @project_status_report  = ProjectStatusReport.new(project: @project)
    @project_revenue_report = ProjectRevenueReport.new(project: @project, grouped: 'monthly')
  end
end
```

```haml
.span6
  = render template: 'dossier/reports/show', locals: {report: @project_status_report.run}
.span6
  = render template: 'dossier/reports/show', locals: {report: @project_revenue_report.run}
```

### Dossier for APIs

```ruby
class Api::ProjectsController < Api::ApplicationController

  def snapshot
    render json: ProjectStatusReport.new(project: @project).results.hashes
  end
end
```

## Advanced Usage

To see a report with all the bells and whistles, check out `spec/sample/app/reports/employee_report.rb` or other reports in `spec/sample/app/reports`.

## Compatibility

Dossier currently supports all databases supported by ActiveRecord; it comes with `Dossier::Adapter::ActiveRecord`, which uses ActiveRecord connections for escaping and executing queries. However, as the `Dossier::Adapter` namespace implies, it was written to allow for other connection adapters. See `CONTRIBUTING.md` if you'd like to add one.

## Protecting Access to Reports

You probably want to provide some protection to your reports: require viewers to be logged in, possibly check whether they're allowed to access this particular report, etc.

Of course, you can protect your own controllers' use of Dossier reports however you wish. To protect report access via `Dossier::Controller`, you can make use of two facts:

1. `Dossier::Controller` subclasses `ApplicationController`
2. If you use an initializer, you can call methods on `Dossier::Controller`

So for a very simple, roll-your-own solution, you could do this:

```ruby
# config/initializers/dossier.rb
Rails.application.config.to_prepare do
  # Define `#my_protection_method` on your ApplicationController
  Dossier::ReportsController.before_filter :my_protection_method
end
```

For a more robust solution, you might make use of some gems. Here's a solution using [Devise](https://github.com/plataformatec/devise) for authentication and [Authority](https://github.com/nathanl/authority) for authorization:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # Basic "you must be logged in"; will apply to all subclassing controllers,
  # including Dossier::Controller.
  before_filter :authenticate_user!
end

# config/initializers/dossier.rb
Rails.application.config.to_prepare do
  # Use Authority to enforce viewing permissions for this report.
  # You might set the report's `authorizer_name` to 'ReportsAuthorizer', and
  # define that with a `readable_by?(user)` method that suits your needs
  Dossier::ReportsController.authorize_actions_for :report_class
end
```

See the referenced gems for more documentation on using them.

## Running the Tests

Note: when you run the tests, Dossier will **make and/or truncate** some tables in the `dossier_test` database.

- Run `bundle`
- `RAILS_ENV=test rake db:create`
- `cp spec/sample/config/database.yml{.example,}` and edit it so that it can connect to the test database.
- `cp spec/fixtures/db/mysql2.yml{.example,}`
- `cp spec/fixtures/db/sqlite3.yml{.example,}`
- `rspec spec`

## Moar Dokumentationz pleaze

- How Dossier uses ORM adapters to connect to databases, currently only AR's are used.
- Examples of connecting to different databases, of the same type or a different one
- Document using hooks and what methods are available in them
- Callbacks, eg:
  - Stored procedures
  - Reformat results
- Linking
  - To other reports
  - To other formats
- Extending the formatter
- Show how to do "crosstab" reports (preliminary query to determine columns, then build SQL case statements?)

## Roadmap

- Make Roadmap

## Versions

Rails 3.x and 4.x are supported in the Dossier 2.x version.
Ruby 2.4+, Rails 4.2.11, Rails 5.0.x, 5.1.x, 5.2.x are supported in the Dossier 3.x version.

