# Index lookups in mutating loops

```toml
[lint]
select = ["PLR1733", "PLR1736"]
```

## Repeated dictionary reads

The value from `.items()` does not track assignments to the dictionary entry. In particular,
replacing the `while` condition would prevent the loop from observing the decremented value.

```py
def palindrome(d, s):
    i, j = 0, len(s) - 1
    op = [""] * len(s)
    for k, v in d.items():
        while d[k] >= 2:
            op[i] = k
            op[j] = k
            i += 1
            j -= 1
            d[k] -= 2
        if d[k]:
            op[len(s) // 2] = k
    return "".join(op)
```

## Changes after a read

Conservatively suppress lookups throughout the body when an assignment or deletion invalidates
the replacement, even if a read appears before the change in source order.

```py
def changes(d, other):
    for k, v in d.items():
        print(d[k])
        d[k] = 0
        print(d[k])

    for k, v in d.items():
        print(d[k])
        d[k]: int = 0

    for k, v in d.items():
        print(d[k])
        d[k] += 1

    for k, v in d.items():
        print(d[k])
        del d[k]

    for k, v in d.items():
        print(d[k])
        d = other

    for k, v in d.items():
        print(d[k])
        k = "other"

    for k, v in d.items():
        print(d[k])
        v = 0

    for k, v in d.items():
        print(d[k])
        if v:
            d[k] = 0

    for k, v in d.items():
        while d[k]:
            k = "other"

    for k, v in d.items():
        while d[k]:
            v = 0
            break
```

## Independent changes

Reads still receive diagnostics when the tracked entry and bindings remain unchanged. An
annotation without a value does not assign to the entry.

```py
def independent_changes(d, other):
    for k, v in d.items():
        print(d[k])  # error: [unnecessary-dict-index-lookup]
        other[k] = 0

    for k, v in d.items():
        print(d[k])  # error: [unnecessary-dict-index-lookup]
        d["other"] = 0

    for k, v in d.items():
        print(d[k])  # error: [unnecessary-dict-index-lookup]
        d[k]: int

    for k, v in d.items():
        while d[k]:  # error: [unnecessary-dict-index-lookup]
            break

    for k, v in d.items():
        print(d[k])  # error: [unnecessary-dict-index-lookup]
        print(d[get_key()])
```

## List lookups share the mutation guard

The same restriction applies to the value yielded by `enumerate()`.

```py
def list_changes(values, other):
    for i, value in enumerate(values):
        while values[i]:
            values[i] -= 1

    for i, value in enumerate(values):
        print(values[i])
        values[i] = 0

    for i, value in enumerate(values):
        print(values[i])
        del values[i]

    for i, value in enumerate(values):
        print(values[i])
        value = 0

    for i, value in enumerate(values):
        print(values[i])  # error: [unnecessary-list-index-lookup]
        other[i] = 0

    for i, value in enumerate(values):
        while values[i]:  # error: [unnecessary-list-index-lookup]
            break
```
