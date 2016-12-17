# URL-Based Permissions

This document describes how to format permissions for users or groups in the following way:

```
<path>?<parameters>:<privileges>
```

Some examples:

- `/groups/my-group:read` - allows read access to the group `/groups/my-group`.
- `/articles?author=user-1:read,update` - allows read and update access to the articles whose author is `user-1`.
- `/**:owner` - allows all possible operations for all resources, e.g. an administrator.
- `https://newspaper.com/articles?author=user1:read,update` - allows read and update access to all newspaper articles whose author is `user1`.

## Goal

URL-Based Permissions are intended to provide authorization for web services that:

1. have a REST API of [maturity level 1](http://martinfowler.com/articles/richardsonMaturityModel.html) or higher;
2. require permissions to be expressed in a succinct way;
3. require resource owners to grant permissions to other users or groups.

## Format

```
<path>?<parameters>:<privileges>
```

An URL Permission consists of three components:

1. **Path**: identifies which resource the permission applies to. Can be an absolute pathname (starting with `/`) or an entire url with domain name and url scheme. Urls may include wildcards `*` and `**` to specify permissions over a range of resource instances.
2. **Parameters**: optional parameters that apply additional restrictions to the permission. For example, a permission `/articles:read` grants read access to all articles whereas `/articles?author=user-1:read` grants read access only to articles whose author is `user-1`.
3. **Privileges**: the privileges allowed on the resource. You can either specify these as a comma-separated set of action names, their abbreviations or an alias (e.g. `create,read,update,delete`, `crud` or `all` which are equivalent).

### Path

URL-Based Permissions are ideal for REST API's. REST URL's are structured as follows:

```
https://api.example.com/collection/:resource_id/sub_collection/:sub_resource2
```

For example;

```
https://api.example.com/articles/article-1/comments/comment-1
```

... the following URL permissions would allow a user to read that comment:

```
/articles:read
/articles/*:read
/articles/*/comments/*:read
https://api.example.com/articles/article-1/comments/comment-1:read
```

Note the subtle difference between `/articles:read` and `/articles/*:read`. The former grants permission to both read and search articles. The latter allows the user only to read articles, not search them.

### Parameters

Parameters are domain-specific properties that restrict a permission. They are similar to how you'd filter a resource collection, for example:

```
https://newspaper.com/api/articles?author=user-1
```

... returns newspaper articles written by `user-1`. URL Permission parameters are used in the same way:

```
/articles?author=user-1:all
```

... which grants access to all CRUD operations on articles written by author `user-1`,

```
/articles?author=user-1&status=published:read
```

... grants read access to published articles of `user-1`,

```
/articles/*/comments?article.status=published:read
```

... grants read access to comments of all published articles (but not read the articles themselves).

### Privileges

Privileges specify which operations are allowed on a resource.

You can fully customize all action identifiers, names and what they represent. The following are the default:

Name      | Identifier | Description
----------|------------|-----------------
`read`    |        `r` | View resource.
`create`  |        `c` | Create resource.
`update`  |        `u` | Update resource.
`delete`  |        `d` | Delete resource.
`manage`  |        `m` | Set permissions of users or groups without `manage` or `super` permission for the applicable resource.
`super`   |        `s` | Set permissions of all users and groups for the applicable resource.

In addition to the privileges above the following aliases exist:

Name      | Alias for | Description
----------|-----------|-------------------
`all`     |    `crud` | Allows user to perform the most common operations on a resource apart from changing user or group permissions.
`manager` |   `crudm` | Allows user to perform CRUD operations and set permissions of users without `manager` or `owner` permissions.
`owner`   |   `cruds` | Allows all possible operations on resource.

## Domain model design

Where to store, link and/or retrieve permissions depends on your project's context. A variety of domain objects are commonly used for this purpose:

* **User**: models a real person.
* **Group**: models a group of users.
* **Account**: models a user or group representation identified by a human-readable identifier such as a username or organization name. Often a user and account are the same object.
* **Role**: models a set of privileges for a user, account or group.
* **Policy**: a document modeling who is allowed to do what in which circumstances.
* *resource instance*: directly attach permissions to resource instance objects (similar to an ACL).

There is no right or wrong in choosing how to store URL Permissions. But here are some tips:

* Roles and groups are often used interchangeably; both represent a collection of users with permissions. Using both roles and groups in the same domain might get confusing.
* Roles are similar to action aliases (see above). Make a clear distinction between the two or use only one of them, not both.
* Accounts might be transferrable from one person to another and should only be linked to permissions that are automatically transferred with it.
* Permissions you want to have removed when a person is removed (say an employee leaves the company) are best attached to the user object.
* Try to avoid directly attaching permissions to resource instance objects and spreading your permissions everywhere. If you feel the need to do this consider using URL Permission parameters or switch to ACL's.

## Other access control models

### Role-Based Access Control (RBAC)

[Role-Based Access Control (RBAC)](https://en.wikipedia.org/wiki/Role-based_access_control) is a centrally administered permission system based on roles. Imagine a newspaper website where employees are granted the following roles:

- **writer**: allowed to write articles and submit them to an editor for publication,
- **editor**: allowed to make changes to articles and publish them,
- **graphics_artist**: allowed to change any visual asset.

The main advantage of RBAC is its simplicity, regardless of the number of users there are only a few well-defined roles and granting permissions is a simple task of assigning the user their correct role(s). Many business domains fit this model well as business processes often have clearly defined roles and responsibilities.

Drawbacks of RBAC are its coarseness and inflexibility. Imagine *writer1* asking *writer2* to review a draft of its article. Drafts are only viewable for authors and editors; there is no clear-cut way to grant `writer2` access to `writer1`'s article using roles only. Creating a new role for the specific purpose of reading and updating a specific article defeats the whole idea of roles being easy to assign and manage.

In this example the following URL-Based Permissions could be defined:

- **writer**: `/articles?author=user-1:owner`
- **editor**: `/articles:all`
- **graphics_artist**: `/assets:all`
- **writer** granted access to review specific article: `/articles/a-great-story:read,update`
- **public users**: `/articles?status=published:read`

### Discretionary Access Control (DAC)

[Discretionary Access Control (DAC)](https://en.wikipedia.org/wiki/Discretionary_access_control) allows users to set permissions using an Access Control List (ACL) for each resource instance they own. An entry on the ACL contains who is allowed to do what for which resource instance. E.g `john-doe` is allowed to `get,create,update` on resource class `article` with object id `g0qv41fl`.

DAC allows fine-grained access control and enables users to manage permissions themselves rather than a restricted set of privileged administrators. In our newspaper example it is trivial for *writer1* to grant read access to a draft article to *writer2* using DAC.

Compared to RBAC managing permissions for each and every resource instance quickly grows a huge unmanageable permission database and possibly technical performance issues. Often groups or roles are used to improve manageablility when using DAC.

Not only is DAC difficult to manage, it cannot be expressed in authorization grants effectively (for example in OAuth scopes).

### Attribute-Based Access Control (ABAC)

[Attribute-Based Access Control (ABAC)](https://en.wikipedia.org/wiki/Attribute-Based_Access_Control) uses policies to define a set of rules that must apply to get access to a resource. For example `IF user has role "editor" OR user is resource owner THEN allow read/write access`.

In advanced systems such policies are described in a custom [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) allowing great flexibility.

ABAC's flexibility comes at a price of high complexity to the point of frustration (yes [AWS IAM Policies](http://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html), I'm looking at you...). Developing your own ABAC DSL can be a huge undertaking.

Many systems require some advanced authorization logic not captured in their main authorization model but express this in code rather than policies.

### The real world

In most web applications a combination of various methods are used to authorize users. In our `newspaper.com` example a pragmatic approach would be to:

- use roles (`editor`, `writer`, ...) to easily centrally administer common permissions of users (RBAC-like),
- allow `writers` to grant extra permissions to other `writers` for the purpose of reviewing drafts (DAC-like),
- allow public users to access articles with status `published` only (ABAC-like).

URL-Based Permissions focusses primarily on the representation of a permission, not what you'd like to do with them. Compared to the other models URL-Based Permissions are:

- not as simple or manageable as roles;
- not as fine-grained as an ACL;
- not as flexible or advanced as policies.

As a result when using URL-Based Permissions you may find yourself using other models as well. However, for most REST web services the model should fit well.

#### Example

Imagine storing a file in a cloud service. By default the upload is private and an URL permission granting access to it may look like:

```
/files/logo.png:all
```

Say you want to grant read-access to this file for anonymous users. To model this in your system you might create a group "anonymous" and add the permission `/files/logo.png:read` to that group. However, allowing anyone to add permissions to the group "anonymous" will quickly grow that group's permission list to millions of records creating performance and manageability issues.

Alternatively, you might create an additional resource url `/public/logo.png` acting as a symlink to `/files/logo.png`. This way by granting a group "anonymous" the permission `/public/*:read` allows anyone to access your publicly shared files. Revoking that access is as simple as removing the symlink. While this works, you might not want to create symlinks like that.

Yet another alternative is to add an attribute `public=true|false` to each file record and check programmatically whether a user is granted access or not. To express this in an URL Permission you can use `/files?public=true:read`.

While URL Permissions are flexible, it is likely you will find yourself in a situation where other access control models are easier to implement and maintain.

# Motivation

The need for URL-Based Permissions originated from building a REST API in a [Microservice Architecture](http://www.martinfowler.com/articles/microservices.html). One of the many disadvantages of a microservice architecture is the complexity of an authorization mechanism. In our [OAuth2](https://oauth.net/2/) authentication service we adopted [JWT's](https://jwt.io/introduction/) to transmit authorization grants safely. JWT's are great; [OAuth2 scopes](https://tools.ietf.org/html/rfc6749#section-3.3) however are a bit vague. The OAuth2 RFC does not specify how to format authorization grants (for good reasons). The URL-Based Permission format is one way to deal with this problem.
