## Experimental: Do If rules

This is experimental feature and represents an advanced version of `match_fields`.
The Do If rules are a tree of nodes. The tree is stored in the Do If Checker instance.
When Do If Checker's Match func is called it calls to the root Match func and then
the chain of Match func calls are performed across the whole tree.

### Node types
**`FieldOp`** Type of node where matching rules for fields are stored.

<br>

**`LengthCmpOp`** Type of node where matching rules for byte length and array length are stored

<br>

**`LogicalOp`** Type of node where logical rules for applying other rules are stored.

<br>


### Field op node
DoIf field op node is considered to always be a leaf in the DoIf tree.
It contains operation to be checked on the field value, the field name to extract data and
the values to check against.

Params:
  - `op` - value from field operations list. Required.
  - `field` - name of the field to apply operation. Required.
  - `values` - list of values to check field. Required non-empty.
  - `case_sensitive` - flag indicating whether checks are performed in case sensitive way. Default `true`.
    Note: case insensitive checks can cause CPU and memory overhead since every field value will be converted to lower letters.

Example:
```yaml
pipelines:
  tests:
    actions:
      - type: discard
        do_if:
          op: suffix
          field: pod
          values: [pod-1, pod-2]
          case_sensitive: true
```


### Field operations
**`Equal`** checks whether the field value is equal to one of the elements in the values list.

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: equal
          field: pod
          values: [test-pod-1, test-pod-2]
```

result:
```
{"pod":"test-pod-1","service":"test-service"}   # discarded
{"pod":"test-pod-2","service":"test-service-2"} # discarded
{"pod":"test-pod","service":"test-service"}     # not discarded
{"pod":"test-pod","service":"test-service-1"}   # not discarded
```

<br>

**`Contains`** checks whether the field value contains one of the elements the in values list.

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: contains
          field: pod
          values: [my-pod, my-test]
```

result:
```
{"pod":"test-my-pod-1","service":"test-service"}     # discarded
{"pod":"test-not-my-pod","service":"test-service-2"} # discarded
{"pod":"my-test-pod","service":"test-service"}       # discarded
{"pod":"test-pod","service":"test-service-1"}        # not discarded
```

<br>

**`Prefix`** checks whether the field value has prefix equal to one of the elements in the values list.

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: prefix
          field: pod
          values: [test-1, test-2]
```

result:
```
{"pod":"test-1-pod-1","service":"test-service"}   # discarded
{"pod":"test-2-pod-2","service":"test-service-2"} # discarded
{"pod":"test-pod","service":"test-service"}       # not discarded
{"pod":"test-pod","service":"test-service-1"}     # not discarded
```

<br>

**`Suffix`** checks whether the field value has suffix equal to one of the elements in the values list.

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: suffix
          field: pod
          values: [pod-1, pod-2]
```

result:
```
{"pod":"test-1-pod-1","service":"test-service"}   # discarded
{"pod":"test-2-pod-2","service":"test-service-2"} # discarded
{"pod":"test-pod","service":"test-service"}       # not discarded
{"pod":"test-pod","service":"test-service-1"}     # not discarded
```

<br>

**`Regex`** checks whether the field matches any regex from the values list.

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: regex
          field: pod
          values: [pod-\d, my-test.*]
```

result:
```
{"pod":"test-1-pod-1","service":"test-service"}       # discarded
{"pod":"test-2-pod-2","service":"test-service-2"}     # discarded
{"pod":"test-pod","service":"test-service"}           # not discarded
{"pod":"my-test-pod","service":"test-service-1"}      # discarded
{"pod":"my-test-instance","service":"test-service-1"} # discarded
{"pod":"service123","service":"test-service-1"}       # not discarded
```

<br>


### Logical op node
DoIf logical op node is a node considered to be the root or an edge between nodes.
It always has at least one operand which are other nodes and calls their checks
to apply logical operation on their results.

Params:
  - `op` - value from logical operations list. Required.
  - `operands` - list of another do-if nodes. Required non-empty.

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: and
          operands:
            - op: equal
              field: pod
              values: [test-pod-1, test-pod-2]
              case_sensitive: true
            - op: equal
              field: service
              values: [test-service]
              case_sensitive: true
```


### Logical operations
**`Or`** accepts at least one operand and returns true on the first returned true from its operands.

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: or
          operands:
            - op: equal
              field: pod
              values: [test-pod-1, test-pod-2]
            - op: equal
              field: service
              values: [test-service]
```

result:
```
{"pod":"test-pod-1","service":"test-service"}   # discarded
{"pod":"test-pod-2","service":"test-service-2"} # discarded
{"pod":"test-pod","service":"test-service"}     # discarded
{"pod":"test-pod","service":"test-service-1"}   # not discarded
```

<br>

**`And`** accepts at least one operand and returns true if all operands return true
(in other words returns false on the first returned false from its operands).

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: and
          operands:
            - op: equal
              field: pod
              values: [test-pod-1, test-pod-2]
            - op: equal
              field: service
              values: [test-service]
```

result:
```
{"pod":"test-pod-1","service":"test-service"}   # discarded
{"pod":"test-pod-2","service":"test-service-2"} # not discarded
{"pod":"test-pod","service":"test-service"}     # not discarded
{"pod":"test-pod","service":"test-service-1"}   # not discarded
```

<br>

**`Not`** accepts exactly one operand and returns inverted result of its operand.

Example:
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: not
          operands:
            - op: equal
              field: service
              values: [test-service]
```

result:
```
{"pod":"test-pod-1","service":"test-service"}   # not discarded
{"pod":"test-pod-2","service":"test-service-2"} # discarded
{"pod":"test-pod","service":"test-service"}     # not discarded
{"pod":"test-pod","service":"test-service-1"}   # discarded
```

<br>


### Length comparison op node
DoIf length comparison op node is considered to always be a leaf in the DoIf tree like DoIf field op node.
It contains operation that compares field length in bytes or array length (for array fields) with certain value.

Params:
  - `op` - must be `byte_len_cmp` or `array_len_cmp`. Required.
  - `field` - name of the field to apply operation. Required.
  - `cmp_op` - comparison operation name (see below). Required.
  - `value` - integer value to compare length with. Required non-negative.

Example 1 (byte length comparison):
```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: byte_len_cmp
          field: pod_id
          cmp_op: lt
          value: 5
```

Result:
```
{"pod_id":""}      # discarded
{"pod_id":123}     # discarded
{"pod_id":12345}   # not discarded
{"pod_id":123456}  # not discarded
```

Example 2 (array length comparison):

```yaml
pipelines:
  test:
    actions:
      - type: discard
        do_if:
          op: array_len_cmp
          field: items
          cmp_op: lt
          value: 2
```

Result:
```
{"items":[]}         # discarded
{"items":[1]}        # discarded
{"items":[1, 2]}     # not discarded
{"items":[1, 2, 3]}  # not discarded
{"items":"1"}        # not discarded ('items' is not an array)
{"numbers":[1]}      # not discarded ('items' not found)
```

Possible values of field `cmp_op`: `lt`, `le`, `gt`, `ge`, `eq`, `ne`.
They denote corresponding comparison operations.

| Name | Op |
|------|----|
| `lt` | `<` |
| `le` | `<=` |
| `gt` | `>` |
| `ge` | `>=` |
| `eq` | `==` |
| `ne` | `!=` |

<br>*Generated using [__insane-doc__](https://github.com/vitkovskii/insane-doc)*