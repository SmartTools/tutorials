---
layout: page
title: Documentation Builder
parent: Third-party
nav_order: 1
---

# Documentation builder

Because the actor based server do not tend to be a classic RESTfull application, we found it unsuitable to use either [Swagger](https://swagger.io/), [API blueprint](https://apiblueprint.org/), etc.

We recommend to use simple markdown documentation for each actor and chain. For generating html pages with this documentation you can use third-party utility [actors-doc-builder](https://github.com/7bits/doc_builder).

## How to use doc-builder

Download it **[here](https://github.com/7bits/doc_builder/releases)**

First of all check whether your project corresponds to [the canonical project structure]({{"/quickstart/project_structure" | relative_url}}) or not.

In every feature you coulds create a special file `doc.md`. When you run in the project root the doc-builder it compiles all markdowns to the static html site that you could deploy.

Usage:

```console
java -jar doc-builder.jar
```

## Best practices

Because we do not have any limitation on the documentation format, we have to arrange a way of writing it.

I recommend use this structure:

- Chain name(address)
- Chain description
- Request [json schema](http://json-schema.org/)
- Request examples
- Response [json schema](http://json-schema.org/)
- Response examples

Example:

```markdown
# Chain `registration`

Register a user to the application with login/password.

## Request schema

` ``json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "registration_request",
  "type": "object",
  "properties": {
    "messageMapId": {
      "type": "string"
    },
    "login": {
      "type": "string",
      "description": "User's login, for example email"
    },
    "password": {
      "type": "string",
      "description": "User's password"
    }
  }
}
` ``

## Request example

` ``json
{
  "messageMapId": "registration",
  "login": "name@example.com",
  "password": "qwerty"
}
`` `

## Response schema

` ``json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "registration_response",
  "type": "object",
  "properties": {
    "success": {
      "type": "boolean",
      "description": "if there is no error return true"
    },
    "error": {
      "type": "string",
      "description": "error message"
    }
  }
}
`` `

## Response example

` ``json
{
  "success": true,
  "error": null
}
`` `

` ``json
{
  "success": "false",
  "error": "User login or password is wrong"
}
` ``

```
