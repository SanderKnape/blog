GitHub Actions documentation doesn't contain a lot of examples.

For example, in the [create event](https://help.github.com/en/actions/reference/events-that-trigger-workflows#create-event-create) documentation you can see that it can trigger either on the creation of a branch, or on the creation of a tag. But the documentation doesn't contain an example, for example how to distinguish between the two.

Also, the documentation regarding [expressions](https://help.github.com/en/actions/reference/contexts-and-expression-syntax-for-github-actions) doesn't contain many examples on how to use the `if:` property to add conditionals to your workflows.

Therefore I compiled a list of useful things that I wasn't able to find easily in the documentation. Is useful to me, so could definitely useful to others.

## Create event

Branch vs tag:

In the GitHub context (`${{ toJson(github) }}`):

`.event.ref = new` for both

`.event.ref_type = tag || branch` <-- so here is the difference

With conditional:

```yaml
jobs:
  one:
    runs-on: ubuntu-16.04
    if: github.event.ref_type == 'branch'
    steps:
      - name: onbranch
        if: github.event.ref_type == 'branch'
        run: echo "On Branch!"
      - name: ontag
        if: github.event.ref_type == 'tag'
        run: echo "On Tag!"
      - name: Dump GitHub context
```

The environment variable `GITHUB_REF` contains for example the value `refs/tags/newnew` when a tag `newnew` is created.

The following does NOT seem to work to trigger on only the creation of tags (and not branches):

```yaml
on:
  create:
    tags:
    - '*'
```

## Push vs. Create

The difference between:

```yaml
on:
  push:
    tags:
      - '*'
```

And:

```yaml
on:
  create:
    tags:
      - '*'
```
