- Start Date: 2019-03-05
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/459
- Tracking: (leave this empty)

# Angle Bracket Invocations For Built-in Components

## Summary

[RFC #311](./0311-angle-bracket-invocation.md) introduced the angle bracket
component invocation syntax. Many developers in the Ember community has since
adopted this feature with very positive feedback. This style of component
invocation will beomce the default style in the Octane edition and become the
primary way component invocations are taught.

However, Ember ships with three built-in components – `{{link-to}}`, `{{input}}`
and `{{textarea}}`. To date, it is not possible to invoke them with the angle
bracket syntax due to various API mismatches and implementation details.

This RFC proposes some small amendments to these APIs and their implementations
to allow them to be invoked with the angle bracket syntax, i.e. `<LinkTo>`,
`<Input>` and `<TextArea>`.

## Motivation

As mentioned above, this will allow Ember developers to invoke components with
a consistent syntax, which should make it easier to teach.

This RFC _does not_ aim to "fix" issues or quirks with the existing APIs – it
merely attempts to provide a way to do the equivilant invocation in angle
bracket syntax.

## Detailed design

### `<LinkTo>`

There are two main problem with `{{link-to}}`:

* It uses positional arguments as the main API.
* It supports an "inline" form (i.e. without a block).

In the new world, components are expected to work with named arguments. This is
both to improve clarity and to match the HTML tags model (which angle bracket
invocations are loosely modelled after). Positional arguments are reserved for
"control-flow-like" components (e.g. `liquid-if`) and to be paired with the
curly bracket invocation style. Since links are not that, it is not appropiate
for this component to use positional params.

When invoked with a block, the first argument is the route to navigate to. We
propose to name this argument explicitly, with `@route`:

```hbs
{{#link-to "about"}}About Us{{/link-to}}

...becomes...

<LinkTo @route="about">About Us</LinkTo>
```

The second argument can be used to provide a model to the route. We propose to
name this argument explicitly, with `@model`:

```hbs
{{#let this.model.posts.firstObject as |post|}}

  {{#link-to "post" post}}Read {{post.title}}...{{/link-to}}

  ...becomes...

  <LinkTo @route="post" @model={{post}}>Read {{post.title}}...</LinkTo>

{{/let}}
```

In fact, it is possible to pass multiple models to deeply nested routes with
additional positional arguments. For this use case, we propose the `@models`
named argument which accepts an array:

```hbs
{{#let this.model.posts.firstObject as |post|}}
  {{#each post.comments as |comment|}}

    {{#link-to "post.comment" post comment}}
      Comment by {{comment.author.name}} on {{comment.date}}
    {{/link-to}}

    ...becomes...

    <LinkTo @route="post.comment" @models={{array post comment}}>
      Comment by {{comment.author.name}} on {{comment.date}}
    </LinkTo>

  {{/each}}
{{/let}}
```

The singular `@model` argument is a special case of `@models`, provided as a
convenience for the common case. Passing both `@model` and `@models` will be an
error. (So would passing insufficient amount of models for the given route, as
it already is today.)

It is also possible to pass query params to the `{{link-to}}` component with
the somewhat awkward `(query-params)` API. We propose to replace it with a
`@query` named argument that simply take a regular hash (or POJO):

```hbs
{{#link-to "posts" (query-params direction="desc" showArchived=false)}}
  Recent Posts
{{/link-to}}

...becomes...

<LinkTo @route="posts" @query={{hash direction="desc" showArchived=false}}>
  Recent Posts
</LinkTo>
```

Finally, as mentioned above, `{{link-to}}` supports an "inline" form without a
block. This form doesn't bring much value and used to cause confusion around
the ordering of the arguments. We propose to simply not support this for the
angle bracket invocation style:

```hbs
{{link-to "About Us" "about"}}

...becomes...

<LinkTo @route="about">About Us</LinkTo>
```

Other APIs of this compoment are already based on named arguments.

#### Migration Path

We would provide a codemod to convert the old invocation style into the new
style.

#### Deprecations

Even though the angle bracket invocation style is recommended going forward
(and should be enforced via a template lint), components can generally be
invoked using the either the curly or angle bracket syntax. Therefore, while
not recommended, `{{link-to}}` would still work and invoke the same component.

In the case that the curly invocation style is used, we propose to issue the
following deprecation warnings:

* Passing positional arguments to `{{link-to}}` (pass named arguments instead)
* Using the inline form of `{{link-to}}` (use the block form instead)
* The `query-params` helper (pass a hash/POJO to the `query` argument instead)

### `<Input>`

Today, the `{{input}}` component is internally implemented as several internal
components that are selected based on the `type` argument. This is intended as
an internal implmentation detail, but as a result, it is not possible to invoke
the component with `<Input>` since it does not exist as a "real" component.

We propose to change this internal implementation strategy to make it possible
to invoke this with angle brackets just like any other components.

For example:

```hbs
{{input type="text" value=this.model.name}}

...becomes...

<Input @type="text" @value={{this.model.name}} />
```

Another example:

```hbs
{{input type="checkbox" name="email-opt-in" checked=this.model.emailPreference}}

...becomes...

<Input @type="checkbox" @name="email-opt-in" @checked={{this.model.emailPreference}} />
```

#### Migration Path

We would provide a codemod to convert the old invocation style into the new
style.

#### Deprecations

None.

### `<TextArea>`

Due to a similar implementation issue, it is also not possible to invoke the
`{{textarea}}` component with angle bracket invocation style.

We propose to change this internal implementation strategy to make it possible
to invoke this with angle brackets just like any other components. However, to
align with the name used in [RFC #176](./0176-javascript-module-api.md), we
propose to take this opportunity to rename the component into `<TextArea>` (as
opposed to `<Textarea>`).

For example:

```hbs
{{textarea value=this.model.body}}

...becomes...

<TextArea @value={{this.model.body}} />
```

#### Migration Path

We would provide a codemod to convert the old invocation style into the new
style.

#### Deprecations

To prevent surprises, `<Textarea>` will also invoke the same component but with
a deprecation warning (use `<TextArea>` instead).

## How we teach this

Going forward, we will focus on teaching the angle bracket invocation style as
the main (only?) way of invoking components. In that world, there wouldn't be
anything extra to teach, as the invocation style proposed in this RFC is not
different from any other components, which is the purpose of this proposal. Of
course, the APIs of these components will still need to be taught, but that is
not a new change.

The only caveat is that, since the advanced `<LinkTo>` APIs require passing
arrays and hashes, the `{{array}}` and `{{hash}}` helper would have to be
taught before those advanced features can be introduced. However, since the
basic usage (linking to top-level routes) does not require either of those
helpers, it doesn't really affect things from a getting started perspective.

It should also be mentioned that, other built-ins, such as `{{yield}}`,
`{{outlet}}`, `{{mount}}`, etc are considered "keywords" not components, they
are also "control-flow-like", so it wouldn't be appropiate to invoke them with
angle brackets.

## Drawbacks

None.

## Alternatives

* `<LinkTo>` could only support `@models` without special casing `@model` as a
  convenience.

* `<LinkTo>` could support a `@text` argument for inline usage.

* `<TextArea>` could just be named `<Textarea>`.

## Unresolved questions

None.