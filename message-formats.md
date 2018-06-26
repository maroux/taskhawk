---
permalink: /message-formats/
---

# Message Format

Message format versions for Taskhawk are described here.

## v1

```
{
    // unique message identifier uuid v4 for every
    // instance of a new message
    "id": "b1328174-a21c-43d3-b303-964dfcc76efc",

    "metadata": {
        // taskhawk priority
        "priority": "high",

        // message creation timestamp. Either
        // epoch milli-seconds, or ISO 8601 string
        "timestamp": 1460868253255,

        // format version for message payload
        "version": "1.0"
    },

    // user defined arbitrary headers (string values)
    "headers": {
        ...
    },

    // name of the task
    "task": "tasks.send_email",

    // required args (optional). May be any valid 
    // JSON value.
    "args": [
        "email@automatic.com",
        "Hello!"
    ],

    // optional args (optional). May be any valid
    // JSON value.
    "kwargs": {
        "from_email": "spam@example.com"
    }
}
```
