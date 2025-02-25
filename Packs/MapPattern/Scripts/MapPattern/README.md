This transformer will take in a value and transform it based on multiple condition expressions (wildcard, regex, etc) defined in a JSON dictionary structure. The key:value pair of the JSON dictionary should be:

"condition expression": "desired outcome"

For example:

```
    {
        ".*match 1.*": "Dest Val1",
        ".*match 2.*": "Dest Val2",
        ".*match 3(.*)": "\\1",
        "*match 4*": {
            "algorithm": "wildcard",
            "output": "Dest Val4"
        }
    }
```

The transformer will return the value matched to a pattern following to the priority.
When unmatched or the input value is structured (dict or list), it will simply return the input value.

## Script Data
---

| **Name** | **Description** |
| --- | --- |
| Script Type | python3 |
| Tags | transformer, string |

## Inputs
---

| **Argument Name** | **Description** |
| --- | --- |
| value | The value to modify. |
| mappings | A JSON dictionary or list of it that contains key:value pairs that represent the "Condition":"Outcome". |
| algorithm | The default algorithm for pattern match. Available algorithm: `literalP`, `wildcard`, `regex` and `regmatch`. |
| caseless | Set to true for caseless comparison, false otherwise. |
| priority | The option to choose which value matched to return. Available options: `first_match` (default) and `last_match`. |
| context | \`demisto\` context: Input . \(single dot\) on \`From previous tasks\` to enable to extract the context data. |
| flags | The comma separated flags for pattern matching in regex. `dotall` (s), `multiline` (m), `ignorecase` (i) and `unicode` (u) are supported. This will apply to all the algorithms. |
| compare_fields | Set to true if you want pattern matching for each field, otherwise false. |

## Outputs
---
There are no outputs for this script.


---
## Syntax for `mappings`
    
    mappings ::= pattern-mapping | field-mapping
                 # `field-mapping` must be used when you set `compare_fields` to true. `pattern-mapping` is used if it is not set.

    pattern-mapping ::= list-pattern-mapping | base-pattern-mapping

    list-pattern-mapping ::= List[base-pattern-mapping]

    base-pattern-mapping ::= Dict[pattern, repl]
    
    field-mapping ::= Dict[field-name, pattern-mapping]
 
    field-name ::= str

    pattern ::= str   # The pattern string which depends on the algorithm given to match with the value.
    
    repl ::= output-str | config
    
    output-str ::= str  # The data to replace to the value.
                        # - Backslash substitution on the template string is available in `regex`
                        # - DT syntax (${}) is available when `context` is enabled.
                        # - As DT syntax, `${..}` refers the value given in the inputs, and `${.<name>}` also refers the value property located at the relateve path to it.

    output-any ::= output-str | Any  # The data to replace to the value.
                                     # `null` is the special value to identify the input value given in this transformer.
    
    algorithm ::= "literal" | "wildcard" | "regex" | "regmatch"
    
    comp-fields ::= List[field] | comma-separated-fields
    
    comma-separated-fields ::= str # Comma separated field
    
    config ::= Dict[str, Any]
              
           The structure is:
              {
                  "algorithm": algorithm,               # (Optional) The algorithm to pattern matching.
                  "output": output-any,                 # (Optional) The data to replace to the value by the pattern.
                  "exclude": pattern | List[pattern],   # (Optional) Patterns to exclude in the pattern matching.
                  "next": mappings                      # (Optional) Subsequent conditions to do the pattern matching with the value taken from the output.
              }


---
## Examples

---
Transform a severity name to the corresponding number.

> algorithm: regmatch

> caseless: true

> priority: first_match

> context:

> flags:

> compare_fields:

#### mappings:

    {
        "Unknown": 0,
        "Informational|Info": 0.5,
        "Low": 1,
        "Medium": 2,
        "High": 3,
        "Critical": 4
    }


| **Input** | **Output** |
| --- | --- |
| High | 3 |
| Informational | 1 |
| Info | 1 |
| Abc | Abc |


---
Normalize a human readable phrase to a cannonical name.

> algorithm: wildcard

> caseless: true

> priority: first_match

> context:

> flags:

> compare_fields:

#### mappings:

    {
        "*Low*": "low",
        "*Medium*": "medium",
        "*High*": "high",
        "*": "unknown"
    }


| **Input** | **Output** |
| --- | --- |
| 1 - Low | low |
| Medium | medium |
| high (3) | high |
| infomation | unknown |


---
Remove all the heading "Re:" or "Fw:" from an email subject.

> algorithm: regex

> caseless: true

> priority: first_match

> context:

> flags:

> compare_fields:

#### mappings:

    {
        "( *(Re: *|Fw: *)*)(.*)": "\\3"
    }

| **Input** | **Output** |
| --- | --- |
| Re: Re: Fw: Hello! | Hello! |
| Hello! | Hello! |


---

Extract the user name field from an text in an Active Directory user account format.

> algorithm: regex

> caseless: true

> priority: first_match

> context:

> flags:

> compare_fields:

#### mappings:

    {
        "([^@]+)@.+": "\\1",
        "[^\\\\]+\\\\(.+)": "\\1",
        "[a-zA-Z_]([0-9a-zA-Z\\.-_]*)": null,
        ".*": "<unknown>"
    }


| **Input** | **Output** |
| --- | --- |
| username@domain | username |
| domain\username | username |
| username | username |
| 012abc$ | &lt;unknown&gt; |

---

Extract the user name field from an quoted text in an Active Directory user account format.

> algorithm: regex

> caseless: true

> priority: first_match

> context:

> flags:

> compare_fields:

#### mappings:

    {
        "\"(.*)\"": {
            "output": "\\1",
            "next": {
                "([^@]+)@.+": "\\1",
                "[^\\\\]+\\\\(.+)": "\\1",
                "[a-zA-Z_]([0-9a-zA-Z\\.-_]*)": null,
                ".*": "<unknown>"
            }
        },
        "([^@]+)@.+": "\\1",
        "[^\\\\]+\\\\(.+)": "\\1",
        "[a-zA-Z_]([0-9a-zA-Z\\.-_]*)": null,
        ".*": "<unknown>"
    }


| **Input** | **Output** |
| --- | --- |
| "username@domain" | username |
| username@domain | username |
| "domain\username" | username |
| domain\username | username |
| "username" | username |
| username | username |
| 012abc$ | &lt;unknown&gt; |


---

Extract first name and last name from an email address in `firstname.lastname@domain`, but the format is `lastname.firstname@domain` in some particular domains.

> algorithm: regex

> caseless: true

> priority: first_match

> context:

> flags:

> compare_fields:

#### mappings:

    [
        {
            "([^.]+)\\.([^@]+)@.+": {
              "exclude": ".*@example2.com",
              "output": "\\1 \\2"
            }
        },
        {
            "([^.]+)\\.([^@]+)@.+": "\\2 \\1",
            "([^@]+)@.+": "\\1"
        }
    ]


| **Input** | **Output** |
| --- | --- |
| john.doe@example1.com | john doe |
| doe.john@example2.com | john doe |
| username@example1.com | username |


---

Normalize a date/time text to `YYYY-MM-DD HH:mm:ss TZ`.


> algorithm: regex

> caseless: true

> priority: first_match

> context:

> flags:

> compare_fields:

#### mappings:

    {
        "(\\d{4})-(\\d{2})-(\\d{2})T(\\d{2}):(\\d{2}):(\\d{2})(\\.\\d+)?Z": "\\1-\\2-\\3 \\4:\\5:\\6 GMT",
        "[^,]+, (\\d{1,2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": {
            "output": "\\2",
            "next": {
                "Jan": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-01-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-01-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Feb": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-02-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-02-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Mar": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-03-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-03-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Apr": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-04-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-04-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "May": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-05-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-05-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Jun": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-06-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-06-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Jul": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-07-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-07-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Aug": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-08-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-08-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Sep": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-09-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-09-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Oct": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-10-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-10-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Nov": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-11-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-11-\\1 \\4:\\5:\\6 \\7"
                    }
                },
                "Dec": {
                    "output": null,
                    "next": {
                        "[^,]+, (\\d) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-12-0\\1 \\4:\\5:\\6 \\7",
                        "[^,]+, (\\d{2}) ([^ ]+) (\\d{4}) (\\d{2}):(\\d{2}):(\\d{2}) ([^ ]+)": "\\3-12-\\1 \\4:\\5:\\6 \\7"
                    }
                }
            }
        }
    }


| **Input** | **Output** |
| --- | --- |
| 2021-01-02T01:23:45.010Z | 2021-01-02 01:23:45 GMT |
| 2021-01-02T01:23:45Z | 2021-01-02 01:23:45 GMT |
| Tue, 3 Jun 2008 11:05:30 GMT | 2008-06-03 11:05:30 GMT |



---

Pattern matching for different nodes

> algorithm: wildcard

> caseless: true

> priority: first_match

> context:

> flags:

> compare_fields: true

#### mappings:

    {
        "IP": {
            "127.*": "localhost"
        },
        "Host": {
            "*.local": "localhost",
            "*": "other"
        }
    }


| **Input** | **Output** |
| --- | --- |
| {"IP": "127.0.0.1"} | "localhost" |
| {"Host": "localhost"} | "localhost" |
| {"IP": "192.168.1.1"} | "other" |



---

Make a text with the `value` field corresponding to the `score` field.

> algorithm: regex

> caseless: true

> priority: first_match

> context: .

> flags:

> compare_fields: true

#### mappings:

    {
        "score": {
            "1": "low - ${.value}",
            "2": "medium - ${.value}",
            "3": "hight - ${.value}",
            ".*": "unknown - ${.value}"
        }
    }


| **Input** | **Output** |
| --- | --- |
| {"score": 1, "value": "192.168.1.1"} | "low - 192.168.1.1" |
| {"score": 4, "value": "192.168.1.1"} | "unknown - 192.168.1.1" |

