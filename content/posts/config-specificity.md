+++
title = 'Applying Specificity to Policy Configurations'
date = 2025-10-31T12:09:37-04:00
+++

While working on an earlier version of the [Conforma](https://conforma.dev/) project, I was
presented with an interesting problem regarding policy configuration. The ability to intuitively
express which rules should be included was not quite there yet.

For example, consider the following snippet from a policy configuration:

```yaml
config:
  include:
    - slsa_build_scripted_build
    - attestation_type
  exclude:
    - attestation_type.pipelinerun_attestation_found
```

The `.` character is a package separator. `foo.bar` means the `bar` rule from the `foo` package.
`foo` means all the rules in the `foo` package.

In the example above, all the rules from `slsa_build_scripted_build` are included. But what about
the rules from `attestation_type`?

Even though intuitive is a subjective metric, I like to think that the least surprising behavior of
the configuration above is that all of the rules from `attestation_type` are included, except for
`attestation_type.pipelinerun_attestation_found`.

## CSS Specificity

[MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Specificity) defines CSS
Specificity as:

> Specificity is the algorithm used by browsers to determine the CSS declaration that is the most
> relevant to an element, which in turn, determines the property value to apply to the element. The
> specificity algorithm calculates the weight of a CSS selector to determine which rule from
> competing CSS declarations gets applied to an element.

In practice, this means matching by ID takes precedence over matching by class which in turn takes
precedence over matching by element type, i.e. `ID > class > type`. This allows applying a certain
style to all `<p>` elements but allow the usage of a class to tweak the style for specific
instances. Similarly, applying a style to `<p>` element by ID, as unusual as that may be, allows
overriding any general style.

## Configuration Specificity

The problem that CSS specificity solves is exactly the same problem I was trying to solve in the
policy configuration.

I decided to give it a try.

First, I established the precedence order. `foo.bar > foo > *`. `*` is a special wildcard value
that means _all_ packages.

Next, I implemented the algorithm. This wasn't quite the multi-column approach used in the CSS
specificity algorithm. Instead of columns, I used a simplistic [positional
notation](https://en.wikipedia.org/wiki/Positional_notation). This caused the (intended?) behavior
of allowing cross "column" comparison. A certain specificity occurring 10 times increases in
specificity.

Finally, I moved over to the tests. To my surprise, all the existing tests passed. I then added
tests to cover the new use cases we could now support, and those passed as well.

Given the nature of the policy configuration, two specificity values had to be computed. One for
inclusion and another for exclusion. A rule is included if the inclusion score is higher than the
exclusion score. This clearly has a bias towards exclusion. The thought at the time was that
exclusions are only used when there is a strong reason to do so, giving them priority seemed
correct.

## New Use Cases

As the project evolved, there were additional uses cases we had to take into account. One of them
was the concept of "collections" which represents a _group_ of packages. This fit well with the
existing algorithm. All we had to do was assign the priority order. I decided to use the same
priority as package, e.g. `foo`. Looking back, a different precedence would have made more sense:
`foo.bar > foo > @group > *`.

## Takeaway

When facing an odd problem where a solution is not immediately clear, generalize the problem.
Chances are the more general problem has already been solved. In this case, I was lucky for having
learned about CSS specificity before. At the time, the CSS specificity algorithm really stuck with
me due to its simplicity and how obscure it seemed to be to most frontend developers, even though
they rely on it everyday. It is quite elegant 🎩

