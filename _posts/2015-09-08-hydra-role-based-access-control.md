---
layout: post
title: Role-Based Access Control for Hydra
author: David Chandek-Stark
tags: hydra "access control"
---

[Note: This article is a work in progress.]

Hydra permissions, which are essentially read/write, appear to follow a file-system paradigm. The more recent interest in the W3C [WebAccessControl](http://www.w3.org/wiki/WebAccessControl) ontology in Fedora 4 continues this aproach. Digital object repositories, however, may require a more extensive set of permissions than those which are sufficient for file systems. In order to manage this complexity, we can employ the concept of a "role" -- in addition to *user* and *group* -- to introduce an abstraction layer between those concrete "agents" and permissions.  These roles help to define responsibilities like "curator" or "contributor" which convey a cluster of permissions that may grow and change over time.

### What's the Difference Between a Role and a Group?

The FOAF vocabulary defines the term [Group](http://xmlns.com/foaf/spec/#term_Group) as "a collection of individual agents".  For the purposes of this article we may say that a group contains *persons* (a.k.a. *users*) and/or other groups. Some software projects (such as [hydra-role-management](https://github.com/projecthydra/hydra-role-management)) use the term "role" to mean the same type of construct.

By contrast I am here using the term *role* to designate an assertion of a authorization relation between an *agent* and a *resource*.  The relation may desinate a "responsibility" or a "capability" which translates into a set of concrete permissions.  For example, we may say the the agent "Matthew" has the "Curator" role on the "Special Stuff" collection; Matthew is *responsible* for the curation of the collection which authorizes him to perform many different actions on the collection.  Now, we could choose to create a "Special Stuff Curators" group, put Matthew in it and grant appropriate permissions to the group on the collection. However, that approach presents us with a couple of problems.  First, we may need a lot of groups to accommodate a large number of resources and possible permissions.  Second, and perhaps more importantly, we don't have a cross-cutting layer that we can leverage to align permissions across the system for functionally equivalent groups.  So, for example, if we create "curators" groups for many collections we may not treat those groups consistently even if that is our intention.

With the introduction of roles we can manage both of these issues.  We don't need to create a "curators" group for each collection; instead, we create a "Curator" role, which is reused across the repository whenever we want to grant the "Curator" set of permissions to an agent (user or group) on a resource.  Since we have separated *agents* from *permissions*, we can change the permissions associated with a role without having to update the role assertions persisted in the repository.

### Terminology and Concepts

The terminology of roles is admittedly (and regrettably) overloaded.  Let's establish some definitions.

A **role** expresses a role assertion, which is a typed, scoped relation between an agent and a resource.

- A role is an instance of the `Role` class, having `role_type`, `agent`, and `scope` attributes.
- A **role is "granted"** when an association is made between a role object and a repository object.
- A **role is "revoked"** when as association between a role object and a repository object is removed (deleted).

A **role type** defines a categorical role assertion -- e.g., Curator, Editor, Viewer.

- A role type is an instance of the `RoleType` class.
- Role types are independent (not hierarchical or related by some kind of inheritance).
- Each role type defines the permissions it will convey when asserted through a role assignment.

An **agent** is a entity (person or group) to whom a role is "granted".

- A **Person** agent (e.g., foaf:Person) represents an individual.
- A **Group** agent (e.g., foaf:Group) represents a set of (zero or more) persons.
- A user is represented by one Person agent and zero or more Group agents (of which its person is member).

The **scope** of a role assertion defines the object(s) to which its privileges apply.

- In *resource scope*, the role applies directly to the object which it is associated.
- In *policy scope* the role applies indirectly to the objects *governed by* (via a policy relationship) the object with which the role is associated.  In a Hydra context, policy scope makes sense only on AdminPolicy objects or other administrative objects that enforce control over other objects.

We can represent the relationships between a repository object, a role and its parts, and the permissions the role ultimately conveys:

![Roles Diagram](/images/roles.png)

### Implementation

Our approach is designed to be loosely coupled with the repository implementation; however, it was developed on a Fedora 3-based Hydra stack -- i.e., hydra-head 7.x, active-fedora 7.x and ActiveTriples 0.2.3 (the highest version compatible with the other dependencies).

#### RoleSet

A `RoleSet` is an abstract class representing a collection of role assertions. The principal APIs evoke SQL semantics:

- `grant` - Add a role assertion to the role set.
- `revoke` - Remove a role assert from the role set.

Two additional methods provide convenciences:

- `replace` - Replace the current role assertions with the provided role assertions.
- `revoke_all` - Removes all assertions from the role set.

Finally, `RoleSet` supports serialization to an Array of role assertions (each of which is serialized as a Hash).

`RoleSet` has two concrete subclasses, `PropertyRoleSet` and `DetachedRoleSet`.

**PropertyRoleSet**

The `PropertyRoleSet` class represents a role set bound to an object property, specifically in ActiveFedora 7.x, the property of an `RDFDatastream` (an `ActiveTriples::Term` in ActiveTriples 0.2.3; more recently this class has been renamed `ActiveTriples::Relation`).

**DetachedRoleSet**

A `DetachedRoleSet` class represents a role set "detached" from a repository object context.  The principal usage of a detached role set is to deserialize a previously serialized role set.  We store the serialized property role set of an object in Solr so a Hydra head can evaluate the role assertions for an object without having to request the data from the repository.  Hence, both the ActiveFedora and SolrDocument objects for a given repository object can be handled consistently with respect to role assertions (of course the SolrDocument is effectively read-only).

#### RoleType

`RoleType` is really the key to the role-based approach. It is the connector between a role assertion and the permissions granted to a user. 

{% highlight ruby %}
class RoleType
  attr_reader :title, :description, :permissions
  alias_method :label, :title

  def initialize(title, description, permissions)
    @title = title.freeze
    @description = description.freeze
    @permissions = permissions.freeze
    freeze
  end

  def to_s
    title
  end
end
{% endhighlight %}

Examples of `RoleType` instances:

{% highlight ruby %}
module RoleTypes
  CURATOR = RoleType.new(
    "Curator",
    "The Curator role conveys responsibility for curating a resource " \
    "and delegating responsibilities to other agents.",
    Permissions::ALL
  )

  EDITOR = RoleType.new(
    "Editor",
    "The Editor role conveys reponsibility for managing the content, " \
    "description and structural arrangement of a resource.",
    [ Permissions::READ, Permissions::DOWNLOAD, Permissions::ADD_CHILDREN,
      Permissions::UPDATE, Permissions::REPLACE, Permissions::ARRANGE ]
  )

  METADATA_EDITOR = RoleType.new(
    "MetadataEditor",
    "The Metadata Editor role conveys responsibility for " \
    "managing the description of a resource.",
    [ Permissions::READ, Permissions::DOWNLOAD, Permissions::UPDATE ]
  )

  CONTRIBUTOR = RoleType.new(
    "Contributor",
    "The Contributor role conveys responsibility for adding related " \
    "resources to a resource, such as works to a collection.",
    [ Permissions::READ, Permissions::ADD_CHILDREN ]
  )

  DOWNLOADER = RoleType.new(
    "Downloader",
    "The Downloader role conveys access to the \"master\" file " \
    "(original content bitstream) of a resource.",
    [ Permissions::READ, Permissions::DOWNLOAD ]
  )

  VIEWER = RoleType.new(
    "Viewer",
    "The Viewer role conveys access to the description and \"access\" " \
    "files (e.g., derivative bitstreams) of a resource.",
    [ Permissions::READ ]
  )

end
{% endhighlight %}

#### Agents

So, how is an actual user associated with a role?  As mentioned above, a user of the system is represented by one or more "agents" -- a singular "person" agent and multiple (0-N) "group" agents, or simply "groups".  In our implementation all agents are represented by strings. A person is represented in the form of an email address. A group is represented by a string that is not of the form of an email address; this could be a simple name like "admins", or an LDAP group name, etc.

A role set may be queried for a the list of roles where the agent is one of the user's agents.  Thus, the "effective roles" for a user is determined by finding the matching resource-scoped roles on the object, and merging that set with the matching policy-scoped roles "inherited" by the object through a policy relationship to another object:

{% highlight %}
class EffectiveRoles < SimpleDelegator
  # @param obj [Object] an object that receives :roles and returns a RoleSet
  # @param agents [String, Array<String>] agent(s) to match roles
  # @return [Ddr::Auth::Roles::RoleSetQuery]
  def self.call(obj, agents)
    new(obj).call(agents)
  end

  # @param agents [String, Array<String>] agent(s) to match roles
  # @return [Ddr::Auth::Roles::RoleSetQuery]
  def call(agents)
    ResourceRoles.call(self)
      .merge(InheritedRoles.call(self))
      .agent(agents)
  end
end
{% endhighlight %}
