---
layout: post
title: A Query Builder for Solr
author: David Chandek-Stark
tags: ruby solr
---

Solr queries are expressed as URL query strings which makes them difficult to read and comprehend, not to mention, construct.  In order to encapsulate these query strings into Query objects, I have developed a QueryBuilder with chainable methods that hopefully also makes the query construction process a bit easier.

Instead	of writing this	query string:

	fq={!term f=foo}bar&fq={!term f=spam}eggs&fq=stuff:(dog OR cat OR bird)&rows=1000&fl=id,foo,bar,stuff

or building this hash:

{% highlight ruby %}
{ fq: ["{!term f=foo}bar", "{!term f=spam}eggs", "stuff:(dog OR cat OR bird)"],
  fl: "id,foo,bar,stuff",
  rows: 1000
}
{% endhighlight %}

I'd rather do this:

{% highlight ruby %}
QueryBuilder.build do |query|
  query.where("foo"=>"bar", "spam"=>"eggs", "stuff"=>["dog", "cat", "bird"])
       .fields("id", "foo", "bar", "stuff")
       .limit(1000)
end
{% endhighlight %}

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

### QueryBuilder

{% highlight ruby %}
class QueryBuilder

    def self.build
      builder = new
      yield builder
      builder.query
    end

    def initialize
      @q       = nil
      @fields  = [ ]
      @filters = [ ]
      @sort    = [ ]
      @rows    = nil
    end

    def query
      Query.new.tap do |qry|
        instance_variables.each do |var|
          qry.instance_variable_set(var, instance_variable_get(var))
        end
      end
    end

    def id(pid)
      q QueryClause.id(pid)
      limit 1
    end

    def filter(*fltrs)
      @filters.push *fltrs
      self
    end

    def fields(*flds)
      @fields.push *flds
      self
    end

    def limit(num)
      @rows = num
      self
    end

    def order_by(field, order)
      @sort << [field, order].join(" ")
      self
    end

    def asc(field)
      order_by field, "asc"
    end

    def desc(field)
      order_by field, "desc"
    end

    def q(q)
      @q = q
      self
    end

    protected

    def method_missing(name, *args, &block)
      if Filter.respond_to? name
        return filter Filter.send(name, *args, &block)
      end
      super
    end

end
{% endhighlight %}