---
layout: post
title: A Query Builder for Solr
author: David Chandek-Stark
tags: ruby solr
---

Solr queries are expressed as URL query strings which makes them difficult to read and comprehend, not to mention, construct.  In order to encapsulate these query strings into Query objects, I have developed a QueryBuilder with chainable methods that hopefully also makes the query construction process a bit easier.

Before presenting the QueryBuilder or the Query class, let's look at the basic components of a query, which I have called `QueryValue`, `QueryClause`, and `Filter`.

### QueryValue

`QueryValue` represents the value portion of a `field:value` Solr query clause.

{% highlight ruby %}
class QueryValue

  class << self
    def build(value)
      RSolr.solr_escape(value) # See https://github.com/rsolr/rsolr
    end

    def or_values(*values)
      value = values.map { |v| build(v) }.join(" OR ")
      "(#{value})"
    end
  end

end
{% endhighlight %}

### QueryClause

`QueryClause` represents the `field:value` Solr query "clause" expression.  Note that the class contains only class methods, the principal of which is `build`.

{% highlight ruby %}
class QueryClause

  PRESENT = "[* TO *]"
  TERM = "{!term f=%s}%s"
  BEFORE_DAYS = "[* TO NOW-%sDAYS]"

  class << self
    # Standard Solr query, no escaping applied
    def build(field, value)
      [field, value].join(":")
    end

    def unique_key(value)
      term(UniqueKeyField.instance, value)
    end
    alias_method :id, :unique_key
    alias_method :pid, :unique_key

    def negative(field, value)
      build "-#{field}", value
    end

    def present(field)
      build field, PRESENT
    end

    def absent(field)
      negative field, PRESENT
    end

    def or_values(field, *values)
      build field, QueryValue.or_values(*values)
    end

    def before(field, date_time)
      value = "[* TO %s]" % date_time.to_time.utc.iso8601
      build field, value
    end

    def before_days(field, days)
      value = BEFORE_DAYS % days.to_i
      build field, value
    end

    def term(field, value)
      TERM % [field, value.gsub(/"/, '\"')]
    end
  end
end
{% endhighlight %}

### Filter

This filter is essentially a list of "raw" (unescaped) query "clauses" which are intended for use with the Solr `fq` filter query parameter.  The raw clauses can be passed in directly, or constructed via a `where` method.

{% highlight ruby %}
class Filter

  class << self
    delegate :where, :raw, :before_days, :before, :present, :absent, to: :new
  end

  attr_accessor :clauses

  def initialize
    @clauses = [ ]
  end

  def where(conditions)
    clauses = conditions.map do |field, value|
      if value.respond_to?(:each)
        QueryClause.or_values(field, *value)
      else
        QueryClause.term(field, value)
      end
    end
    raw *clauses
  end

  # Adds clause (String) w/o escaping
  def raw(*clauses)
    self.clauses += clauses
    self
  end

  def present(field)
    raw QueryClause.present(field)
  end

  def absent(field)
    raw QueryClause.absent(field)
  end

  def before(field, date_time)
    raw QueryClause.before(field, date_time)
  end

  def before_days(field, days)
    raw QueryClause.before_days(field, days)
  end

end
{% endhighlight %}  