# API Security service and filter

This module provides a service and a render filter which can be fed by different API providers to secure access. A centralized
configuration file provides rules for APIs associating an operation to a required permission.

## Permission configuration

API can be restricted based on permissions by adding `org.jahia.modules.api.permissions-*.cfg` files into `digital-factory-data/karaf/etc`.
These files contain a list of rules which will be checked for each node potentially returned by the API. Rules coming from all files are merged and sorted. Rules can be based on path, node type and/or workspace. 

A permission check is always done on a JCR node, and is associated with an API name. Different API names are provided for 
json views, the JCRest API module, or the GraphQL API.

The first rule that matches defines the permission that will be checked on the node to ensure the user is correctly allowed to use the API on it. 

A permission rule is defined by a list of entries of the following format :

```
permission.<rulename>.<propertyname>=<value>
```

A rule can define an `access`, `permission`, `requiredPermission`, `scope`, or `requiredScope` property. `access` can take 2 different values : 
- `denied` : Forbidden for everybody
- `restricted` : Only allowed for users who have the `api-access` permission on the node

The check can be defined more precisely by using `requiredPermission` instead : the value is the name of the permission that will be checked if the rule matches. The user must have this permission to be granted access.

Instead of a permission, a rule can specify a `requiredScope`. Scope can be granted to a request through tokens passed in `Authorization` header (see Scope and tokens below). The access will be granted if and only if the scope is owned by the request.

Another option is to specify `permission` or `scope` - if the user has the permission/scope, his access will be granted, and no other rule will be checked. But it's not required - if he does not have it, subsequent rules will be checked. 

By default, if none of these parameters are passed, the API is granted - no other rules will be tested, and no permission needs to be checked.

A rule can specify any number of matching criteria. Each of these criteria can have a single value or a comma separated list of values.
 - `api` : The names of the API, if the rule should only apply to some entry points
 - `pathPattern` : Regular expressions that will be tested on the node path.
 - `workspace` : `live` or `default`, only request on the specified workspace will match.
 - `nodeType` : Only request on nodes of these type will match.
 - `scope` : A list of scopes 
 
An optional priority can be also specified - the rules with the lowest value will be executed first. Default priority is 0.

```
permission.checkFirst.priority=-10
```

A rule with no condition will always match (and thus, should have a very high value for priority so that it is checked last):
```
permission.global.access=restricted
permission.global.priority=99999
```

Rules can specify a simple path pattern : 
```
permission.users.pathPattern=/sites/[^/]+/home/.*
```

Or a combination of all criterias :
```    
permission.digitallPostsInLive.pathPattern=/sites/digitall/.*
permission.digitallPostsInLive.nodeType=jnt:post,jnt:message
permission.digitallPostsInLive.workspace=live
permission.digitallPostsInLive.requiredPermission=jcr:write
```

### Permissions in a module

A module can package a configuration file in META-INF/configurations folder. Since DX version 7.2.2.0, all `cfg` files in this folder are deployed in `karaf/etc` at module startup. This gives the possibility for a module to provide a defaut configuration file that can be edited by the user. The files are never updated nor removed automatically. The file name can contain the name of the module : `org.jahia.modules.api.permissions-<modulename>.cfg`.

## Scope and tokens

A request can be granted a scope through the usage of tokens, passed in the `Authentication: Bearer` header. 

Security filter currently only support signed JWT token. Tokens contain a verified list of scopes, along with restriction on its usage. 
It's possible to restrct the usage of a token based on the client IP or referer header.

Tokens can be generated via the tools section "jwtConfiguration" - the user can specify the list of scopes that will be owned by the token, and fill in the optional restrictions. You must customize org.jahia.modules.jwt.token.cfg configuration file before generating any token. The file contains the following properties :

- `jwt.issuer` : Name of your organization, that will be included in tokens, only for informational purpose
- `jwt.audience` : The target audience is an identifier for you DX installation - audience is included in the token at generation, and only tokens with the same audience will be accepted.
- `jwt.algorithm` : Algorithm used to sign the token. Only HMAC supported.
- `jwt.secret` : Secret key used to be used with HMAC. It will be used to sign and validate tokens. You must change the secret and keep it safe - any token signed with the same secret can be accepted and will grant the associated scopes.

## Render filter

A render filter catches all ajax calls to `*.json` and `*.html.ajax`. The filter calls the service to check if the request is allowed or not.
The API name contains the template type and the name of the view itself : `view.<template type>.<view name>`

So for example, the following rule will apply to all requests on the tree.json view :

```
permission.tree.api=view.json.tree
```

The following rule will match all json views on pages :

```
permission.tree.api=view.json
permission.tree.nodeType=jnt:page
```

## Module

A default configuration file is provided by this module. On DX 7.2.2+ , the configuration file will be copied automatically to `karaf/etc` 
once the module is started. An existing file won't be overwritten, and the configuration is never automatically removed.
On older versions of DX, the configuration file need to be deployed and maintained manually.

