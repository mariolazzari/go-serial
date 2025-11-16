# Effective Serialization In Go: JSON, Protocol Buffers and More

## Serialization overview

### Why serialization

Serialization in Go (and in most programming languages) is needed whenever you want to convert in-memory data (structs, maps, slices, etc.) into a format that can be stored or transmitted, and later reconstructed back into the same data structure.

```go
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
)

func main() {
	printEncoding(int64(12345678))
	printEncoding(2.718281)
	printEncoding("I ♡ Go")
}

func printEncoding(val any) {
	v := val
	if s, ok := v.(string); ok {
		v = []byte(s)
	}

	var buf bytes.Buffer
	if err := binary.Write(&buf, binary.LittleEndian, v); err != nil {
		fmt.Printf("%#v: error - %s\n", val, err)
		return
	}

	fmt.Printf("%#v: %x\n", val, buf.Bytes())
}
```

### Picking a format

Maturity = (Effort + Time) / Complexity

### General rules

- Do not invent
- Do not mix
- Serialize only at the edge of your code
- Valid JSON - Valid Data

### Formats overview

#### Encoding GOB

- Standard library
- Binary
- Wide range of types
- Go only

#### Encoding JSON

- Standard libraty
- Textual
- Limited range of types
- Most languages
- No schema

#### YAML and TOML
