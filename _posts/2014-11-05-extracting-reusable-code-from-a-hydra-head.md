---
layout: post
title: Extracting reuseable code from a Hydra head
author: David Chandek-Stark
tags: hydra refactoring ruby rails models
---

When we started building a repository application with the [Hydra](http://projecthydra.org) framework nigh about two years ago, nearly everything about the code environment was new to us -- including Ruby, Rails, RSpec and Git.  While we had a respectable level of repository domain knowledge (by no means experts), we tended to follow coding and testing patterns we observed within the Hydra development community.

I think we got the conceptual modeling mostly right.  We stuck to models that seemed to satisfy our initial use cases and hoped that they would give us a good framework to refine as we rolled out the repository to a broader audience.  With one significant exception, [this wiki page](https://github.com/duke-libraries/ddr-models/wiki/Repository-Models-1.0) describes our models and the relations between them.  Originally we had a separate AdminPolicy object which implemented the Hydra admin policy behavior; however, because we observed in practice that we had one AdminPolicy object per Collection, we decided to migrate the admin policy functionality to the Collection model.  (In the long term, we are developing a role-based access control approach, but that is another story ...)

In recent months we have begun planning for at least two additional Hydra heads.  This step caused us to embark on a refactoring of our original Hydra head, extracting the repository models into a separate project which we could reuse as a gem.  In general, this process has gone reasonably well -- meaning we're getting through it -- but we were rather naive about how complicated it would be.  A better knowledge of Rails, gems, and refactoring in general would have helped, not to mention more code analysis and planning before diving in.  A few specific issues deserve mention:

### Namespaces (or not)

Following the typical Rails app development pattern, we originally put our repository models in `app/models`, not namespaced.  This created two issues:

- We couldn't namespace the models on refactoring (e.g., from `Component` to `Ddr::Models::Component`) without touching every object in the repository, which we didn't want to do.

- We learned that `app/models` is not a magic path in a Rails engine (more on engines later), which affects Rails autoloading, etc., so we explicitly required each module:

{% highlight ruby %}
Dir[Ddr::Models::Engine.root.to_s + "/app/models/*.rb"].each { |m| require m }
{% endhighlight %}

(We also added `app/models` to the gemspec `require_paths`, but I don't think that's necessary.)

### Hydra rightsMetadata dependency

If you have a true Hydra model, then you have rightsMetadata, which you get from `Hydra::AccessControls::Permissions`.  Unfortunately, this module (as of this writing) is not on a path available to the engine (b/c it depends on a Rails app?).  Here's the hack we came up with:

{% highlight ruby %}
# Awful hack to make Hydra::AccessControls::Permissions accessible
$: << Gem.loaded_specs['hydra-access-controls'].full_gem_path + "/app/models/concerns"
{% endhighlight %}

### Fedora / Solr / Jetty

Testing repository models requires a Java servlet container for Fedora and Solr, for which most folks use Jetty.  To integrate Jetty into the engine it's convenient to use jettywrapper, which provides hydra-jetty and a number of useful rake tasks (jetty:start, jetty:stop, etc.).  An undocumented (or poorly documented) feature of jettywrapper is that it will use a constant `JETTY_CONFIG` for its default configuration, so the relevant parts of the Rakefile look like this:

{% highlight ruby %}
ENGINE_ROOT = File.dirname(__FILE__)

JETTY_CONFIG = { 
  jetty_home: File.join(ENGINE_ROOT, "jetty"),
  jetty_port: 8983,
  startup_wait: 45,
  quiet: true,
  java_opts: ["-Xmx256m", "-XX:MaxPermSize=128m"]
}

require 'jettywrapper'

Jettywrapper.instance.base_path = ENGINE_ROOT

desc "Run the CI build"
task :ci => ["jetty:clean"] do
  Jettywrapper.wrap(JETTY_CONFIG) do
    Rake::Task['spec'].invoke
  end
end
{% endhighlight %}

### Custom Predicates

ActiveFedora provides that custom predicates for RELS-EXT relations (FC3) can be added to a file at `config/predicate_mappings.yml`.  Unfortunately, this file apparently must be read when `ActiveFedora::Predicates` first loads or it has no effect.  Fortunately, you can use `ActiveFedora::Predicates.set_predicates(mapping)` to add predicates later.  So, back in the Rails engine module, we created an initializer:

{% highlight ruby %}
initializer 'ddr_models.predicates' do
  ActiveFedora::Predicates.set_predicates(Ddr::Metadata::PREDICATES)
end
{% endhighlight %}

where our predicates are defined:

{% highlight ruby %}
module Ddr
  module Metadata
    PREDICATES = {
      "http://projecthydra.org/ns/relations#" => {
        is_attached_to: "isAttachedTo"
      },
      "http://www.loc.gov/mix/v20/externalTarget#" => {
        is_external_target_for: "isExternalTargetFor",
        has_external_target: "hasExternalTarget"
      }
    }
  end
end
{% endhighlight %}

### Database migrations

Our repository models have a dependency on an ActiveRecord model that we use for event tracking, so we had to move that model into the gem as well.  In order to test everything, we needed a db with migrations.  While it appears to be technically possible to create something less than a full Rails engine that can deal with database connectivity and migrations (see [Jeremy Friesen's article](http://ndlib.github.io/practices/composing-a-rails-plugin-for-hydra/)), it's also true that a Rails engine with an internal test app makes this easy (mainly because of the [Rails engine rakefile](https://github.com/rails/rails/blob/master/railties/lib/rails/tasks/engine.rake)).  In the long run it would be nice to dynamically generate the test app (say, with engine_cart), but the static test app generated by Rails got us up and running fast.

Of course, since our original app already had the schema and database table for our Event model, we had to be a little more careful in defining the migration to only create the table if it does not exist:

{% highlight ruby %}
class CreateEvents < ActiveRecord::Migration
  def up
    unless table_exists?("events")
      # table definition and indexes copied from original app schema
      create_table "events" do |t|
        t.datetime "event_date_time"
        t.integer  "user_id"
        t.string   "type"
        t.string   "pid"
        t.string   "software"
        t.text     "comment"
        t.datetime "created_at"
        t.datetime "updated_at"
        t.string   "summary"
        t.string   "outcome"
        t.text     "detail"
      end

      add_index "events", ["event_date_time"], name: "index_events_on_event_date_time"
      add_index "events", ["outcome"], name: "index_events_on_outcome"
      add_index "events", ["pid"], name: "index_events_on_pid"
      add_index "events", ["type"], name: "index_events_on_type"
    end
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
{% endhighlight %}

## Lessons Learned

Of course, some things only come with experience ... but here some observations.

- If you think there's a chance you will build more than one hydra head, consider modularizing and namespacing your code from the start.  I suspect that it's much easier to connect pieces than to disentangle them.  You'll also get practice with gems and Rails engines.

- Refactoring is a discipline (and I'm not very good at it).  I tried to keep [Martin Fowler's definition](http://refactoring.com/) in mind, but it's hard to resist all the temptations to "fix stuff" at the same time that you're reorganizing.

- Good quality tests are critical when making extensive organizational changes.  I'm still learning how to write good ones and avoid bad and unnecessary tests.

- Unless you've got way more git-fu that I, you may want to freeze development on (or at least the relevant parts of) the original app while you're extracting code.  I didn't want to think about how I was going to deal with merge conflicts and applying patches across projects.
