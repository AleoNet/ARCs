# parser

## Development

The parser expects each `arc-xxxx/README.md` file to adhere to the following format:
```
---
title: This is a test
description: Once upon a time...
---

# Title
Lorem ipsum...
```

The parser will then parse it into:
```json
{
  metadata: {
    title: "This is a test",
    description: "Once upon a time..."
  },
  content: "# Title\nLorem ipsum..."
}
```
