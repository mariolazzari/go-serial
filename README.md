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

#### GOB

- Standard library
- Binary
- Wide range of types
- Go only

#### JSON

- Standard libraty
- Textual
- Limited range of types
- Most languages
- No schema

#### YAML and TOML

- Not in standard library
- Textual
- Good range of types
- Most languages
- No schema
- Configuration

#### XML

- Standard library
- Textual
- No types
- Most languages

#### CSV

- Standard library
- Textual
- No types
- Most languages
- No schema

#### Protocol buffer

- Not in standard library
- Binary
- Good range of types
- Many languages
- Schema

### SQL

- Standard library
- Binary
- Good range of types
- Many languages
- Schema

## Go specific formats

### String representation

```go
package email

import "fmt"

// Email is a user email.
type Email struct {
	Name    string
	Address string
}

// String implements fmt.Stringer.
func (e Email) String() string {
	return fmt.Sprintf("%s <%s>", e.Name, e.Address)
}
```

### Encoding TextMarshaler

[TextMarshaler](https://pkg.go.dev/encoding#TextMarshaler)

```go
package coord

import (
	"fmt"
	"math"
)

type Coord struct {
	Lat float64
	Lng float64
}

// MarshalText implement encoding.TextMarshaler
func (c Coord) MarshalText() ([]byte, error) {
	lat, lng := math.Abs(c.Lat), math.Abs(c.Lng)
	n := "N"
	if c.Lat < 0 {
		n = "S"
	}

	e := "E"
	if c.Lng < 0 {
		e = "W"
	}

	s := fmt.Sprintf("%.6f%s %.6f%s", lat, n, lng, e)
	return []byte(s), nil
}
```

### Encoding Gob

[gob](https://pkg.go.dev/encoding/gob)

```go
package store

import "time"

// Item is an item in the store.
type Item struct {
	SKU   string
	Name  string
	Price int // In ¢
	Added time.Time
	Image []byte
}
```

### Challenge: user database

```go
package db

import (
	"bytes"
	"encoding/gob"
	"fmt"

	"go.etcd.io/bbolt"
)

type Role string

var (
	RoleReader Role = "reader"
	RoleWriter Role = "writer"
	RoleAdmin  Role = "admin"
)

type User struct {
	Login string
	Roles []Role
}

type DB struct {
	conn *bbolt.DB
}

var (
	bucketName = []byte("users")
)

// Open returns a new database.
func Open(dbPath string) (*DB, error) {
	conn, err := bbolt.Open(dbPath, 0666, nil)
	if err != nil {
		return nil, err
	}

	err = conn.Update(func(tx *bbolt.Tx) error {
		_, err := tx.CreateBucketIfNotExists(bucketName)
		return err
	})
	if err != nil {
		return nil, err
	}

	return &DB{conn}, nil
}

// Close closes the database.
func (db *DB) Close() error {
	return db.conn.Close()
}

// Put puts a new user in the database.
func (db *DB) Put(u User) error {
	var buf bytes.Buffer
	if err := gob.NewEncoder(&buf).Encode(u); err != nil {
		return err
	}

	err := db.conn.Update(func(tx *bbolt.Tx) error {
		b := tx.Bucket(bucketName)
		return b.Put([]byte(u.Login), buf.Bytes())
	})
	if err != nil {
		return err
	}

	return nil
}

// Get gets a user from the database.
func (db *DB) Get(login string) (User, error) {
	var data []byte

	err := db.conn.View(func(tx *bbolt.Tx) error {
		b := tx.Bucket(bucketName)
		data = b.Get([]byte(login))
		return nil
	})

	if err != nil {
		return User{}, err
	}

	if data == nil {
		return User{}, fmt.Errorf("%q - not found", login)
	}

	var u User
	if err := gob.NewDecoder(bytes.NewReader(data)).Decode(&u); err != nil {
		return User{}, err
	}

	return u, nil
}
```

## Working with JSON

### Encoding JSON Api

[json](https://pkg.go.dev/encoding/json)

| Direction  | Via       | Use            |
| ---------- | --------- | -------------- |
| JSON -> Go | []byte    | json.Unmarshal |
| GO -> JSON | []byte    | json.Marshal   |
| JSON -> Go | io.Reader | json.Decoder   |
| GO -> JSON | io.Writer | json.Encoder   |

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
)

type Task struct {
	User     string `json:"user"`
	Priority int    `json:"priority"`
	Payload  []byte `json:"payload"`
}

func main() {
	t := Task{
		User:     "garfield",
		Priority: 1,
		Payload:  []byte("nap"),
	}

	// []byte API
	data, err := json.Marshal(t)
	if err != nil {
		fmt.Println("error (marshal):", err)
		return
	}
	fmt.Println("data:", string(data))

	var t2 Task
	if err := json.Unmarshal(data, &t2); err != nil {
		fmt.Println("error (unmarshal):", err)
		return
	}
	fmt.Println("t2 (unmarshal):", t2)

	// io.Writer/io.Reader API
	var buf bytes.Buffer

	enc := json.NewEncoder(&buf)
	if err := enc.Encode(t); err != nil {
		fmt.Println("error (encode):", err)
		return
	}
	fmt.Println("data:", buf.String())

	dec := json.NewDecoder(&buf)
	if err := dec.Decode(&t2); err != nil {
		fmt.Println("error (decode):", err)
		return
	}
	fmt.Println("t2 (decoder):", t2)
}
```

### Custom marshaling

```go
package date

import (
	"bytes"
	"encoding/json"
	"fmt"
	"time"
)

type Date struct {
	Year  int
	Month time.Month
	Day   int
}

func (d Date) MarshalJSON() ([]byte, error) {
	// Step 1: Convert to type json.Marshal can handle
	s := fmt.Sprintf("%04d-%02d-%02d", d.Year, d.Month, d.Day)

	// Step 2: Use json.Marshal
	return json.Marshal(s)

	// Step 3: There is no step 3 ☺
}

func (d *Date) UnmarshalJSON(data []byte) error {
	var year, month, day int

	r := bytes.NewReader(data)
	if _, err := fmt.Fscanf(r, `"%04d-%02d-%02d"`, &year, &month, &day); err != nil {
		return err
	}

	d.Year, d.Month, d.Day = year, time.Month(month), day
	return nil
}
```

### Zero values vs missing

Go default initialization can cause problems with initialization

- Pointers
- map[string]any
- Setting defaults

```go
package shop

import (
	"encoding/json"
	"fmt"
)

type OrderRequest struct {
	ClientID string `json:"client_id"`
	SKU      string `json:"sku"`
	Amount   int    `json:"amount"`
}

func ParseOrderRequest(data []byte) (OrderRequest, error) {
	req := OrderRequest{
		Amount: 1,
	}
	if err := json.Unmarshal(data, &req); err != nil {
		return OrderRequest{}, err
	}

	if req.Amount <= 0 {
		return OrderRequest{}, fmt.Errorf("invalid amount: %d", req.Amount)
	}

	return req, nil
}
```

### Dynamic types

[Raw message](https://pkg.go.dev/encoding/json#example-RawMessage-Marshal)

```go
package elevator

import (
	"encoding/json"
	"fmt"
)

// Event is the main event type.
type Event struct {
	Type    string
	Payload json.RawMessage
}

// DoorEvent is a door open/close event.
type DoorEvent struct {
	Action string
	Floor  int
}

// ButtonEvent is a button click event.
type ButtonEvent struct {
	Button int
}

// Handle event handles an event.
func HandleEvent(data []byte) error {
	var e Event
	if err := json.Unmarshal(data, &e); err != nil {
		return err
	}

	switch e.Type {
	case "door":
		var d DoorEvent
		if err := json.Unmarshal(e.Payload, &d); err != nil {
			return err
		}
		fmt.Println("door event:", d) // TODO: Handle event
	case "button":
		var b ButtonEvent
		if err := json.Unmarshal(e.Payload, &b); err != nil {
			return err
		}
		fmt.Println("button event:", b) // TODO: Handle event
	default:
		return fmt.Errorf("unknown event - %q", e.Type)
	}

	return nil
}
```

[mapstructure](https://pkg.go.dev/github.com/go-viper/mapstructure/v2)

```go
package elevator

import (
	"encoding/json"
	"fmt"

	"github.com/mitchellh/mapstructure"
)

// DoorEvent is a door open/close event.
type DoorEvent struct {
	Action string
	Floor  int
}

// ButtonEvent is a button click event.
type ButtonEvent struct {
	Button int
}

// Handle event handles an event.
func HandleEvent(data []byte) error {
	var e map[string]any
	if err := json.Unmarshal(data, &e); err != nil {
		return err
	}

	switch e["type"] {
	case "door":
		var d DoorEvent
		if err := mapstructure.Decode(e, &d); err != nil {
			return err
		}
		fmt.Println("door event:", d) // TODO: Handle event
	case "button":
		var b ButtonEvent
		if err := mapstructure.Decode(e, &b); err != nil {
			return err
		}
		fmt.Println("button event:", b) // TODO: Handle event
	default:
		return fmt.Errorf("unknown event - %q", e["type"])
	}

	return nil
}
```

### Streaming JSON

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"time"
)

// Log is a log in the database.
type Log struct {
	Time    time.Time `json:"time"`
	Level   string    `json:"level"`
	Message string    `json:"message"`
}

// Producer agent tailing logs.
func Producer(w io.WriteCloser, n int) error {
	defer w.Close()

	enc := json.NewEncoder(w)
	start := time.Date(2024, time.April, 3, 11, 47, 35, 0, time.UTC)
	delta := 523 * time.Millisecond

	for i := range n {
		log := Log{
			Time:    start.Add(time.Duration(i) * delta),
			Level:   "info",
			Message: fmt.Sprintf("log #%d", i+1),
		}
		if err := enc.Encode(log); err != nil {
			return err
		}
		time.Sleep(100 * time.Millisecond) // Simulate network latency
	}

	return nil
}

// Consumer reads logs from r.
func Consumer(r io.Reader) error {
	dec := json.NewDecoder(r)

	for {
		var log Log
		err := dec.Decode(&log)
		if errors.Is(err, io.EOF) {
			break
		}
		if err != nil {
			return err
		}

		fmt.Println("log:", log) // TODO: Handle logs
	}

	return nil
}
```

### Challenge: metrics

```go
package metrics

import (
	"bytes"
	"encoding/json"
	"fmt"
	"time"
)

type Metric struct {
	Time   time.Time
	Name   string
	Value  float64
	Labels map[string]string
}

// MarshalText implements encoding.TextMarshaler
func (m Metric) MarshalText() ([]byte, error) {
	var buf bytes.Buffer
	fmt.Fprintf(&buf, m.Name)
	if len(m.Labels) > 0 {
		fmt.Fprintf(&buf, "{")
		i := 0
		for k, v := range m.Labels {
			fmt.Fprintf(&buf, "%s=%q", k, v)
			if i < len(m.Labels)-1 {
				fmt.Fprint(&buf, ",")
			}
			i++
		}
		fmt.Fprintf(&buf, "}")
	}
	fmt.Fprintf(&buf, " %f", m.Value)
	if !m.Time.IsZero() {
		fmt.Fprintf(&buf, " %d", m.Time.UnixMicro())
	}

	return buf.Bytes(), nil
}

type Metrics []Metric

func (ms Metrics) MarshalJSON() ([]byte, error) {
	m := map[string]any{
		"count":   len(ms),
		"metrics": []Metric(ms),
	}

	return json.Marshal(m)
}
```

## Protocol buffers

### Protocol buffers overview
