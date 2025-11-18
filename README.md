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

[Docs](https://protobuf.dev/)

- Smchema generates code
- Binary format
- Many languages
- Good ecosystem

```ts
syntax = "proto3";

import "google/protobuf/timestamp.proto";

option go_package = "rides/pb";

message Location {
  double lat = 1;
  double lng = 2;
}

enum RideType {
  UNKNOWN = 0;
  REGULAR = 1;
  SHARED = 2;
}

message RideStart {
  google.protobuf.Timestamp time = 1;
  string car_id = 2;
  Location location = 3;
  RideType type = 4;
  int64 passengers = 5;
}
```

```go
package rides

import (
	"fmt"
	"time"
)

type RideType byte

const (
	RegularType RideType = iota + 1
	SharedType
)

func (t RideType) String() string {
	switch t {
	case RegularType:
		return "regular"
	case SharedType:
		return "shared"
	}

	return fmt.Sprintf("RideType(%d)", t)
}

type Location struct {
	Lat float64
	Lng float64
}

type RideStart struct {
	Time       time.Time
	CarID      string
	Location   Location
	Type       RideType
	Passengers int
}
```

### Generating code

[Go](https://protobuf.dev/getting-started/gotutorial/)

```go
/* Tools

# protoc

Install via package manager (apt, brew, choco ...)

# Go Protocol Buffers Plugin

go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
*/

//go:generate mkdir -p pb
//go:generate protoc --go_out=pb --go_opt=paths=source_relative rides.proto
```

### Using code

```go
package main

import (
	"encoding/json"
	"fmt"

	"google.golang.org/protobuf/proto"
	"google.golang.org/protobuf/types/known/timestamppb"

	"serialization/Ch04/04_03/pb"
)

func main() {
	// Marshal
	msg := pb.RideStart{
		Time:  timestamppb.Now(),
		CarId: "McQueen",
		Location: &pb.Location{
			Lat: 48.8737917,
			Lng: 2.2950275,
		},
		Type:       pb.RideType_REGULAR,
		Passengers: 1,
	}
	fmt.Println("msg :", &msg)

	data, err := proto.Marshal(&msg)
	if err != nil {
		fmt.Println("ERROR: marshal:", err)
		return
	}
	fmt.Printf("proto size: %5d\n", len(data))

	// Compare size to JSON
	jdata, err := json.Marshal(&msg)
	if err != nil {
		fmt.Println("ERROR: json marshal:", err)
		return
	}
	fmt.Printf(" json size: %5d\n", len(jdata))

	// Unmarshal
	var msg2 pb.RideStart
	if err := proto.Unmarshal(data, &msg2); err != nil {
		fmt.Println("ERROR: unmarshal:", err)
		return
	}
	fmt.Println("msg2:", &msg2)
}
```

### Working with time

```go
package main

import (
	"fmt"
	"time"

	"google.golang.org/protobuf/types/known/timestamppb"
)

func main() {
	t := time.Date(2024, 4, 13, 17, 32, 47, 203, time.UTC)
	fmt.Println("t :", t)

	// time.Time -> timestamppb.Timestamp
	pt := timestamppb.New(t)
	fmt.Println("pt:", pt)

	// timestamppb.Timestamp -> time.Time
	t2 := pt.AsTime()
	fmt.Println("t2:", t2)
}
```

### Emitting JSON

```go
package main

import (
	"encoding/json"
	"fmt"
	"serialization/Ch04/04_05/pb"

	"google.golang.org/protobuf/types/known/timestamppb"
)

func main() {
	msg := pb.RideStart{
		Time:  timestamppb.Now(),
		CarId: "McQueen",
		Location: &pb.Location{
			Lat: 48.8737917,
			Lng: 2.2950275,
		},
		Type:       pb.RideType_REGULAR,
		Passengers: 1,
	}

	data, err := json.MarshalIndent(&msg, "", "    ")
	if err != nil {
		fmt.Println("ERROR:", err)
		return
	}
	fmt.Println(string(data))
}
```

### Challenge

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"os"
	"time"

	"serialization/Ch04/solution/pb"

	"google.golang.org/protobuf/proto"
	"google.golang.org/protobuf/types/known/timestamppb"
)

func main() {
	// end ride message
	// id: "aa08deb"
	// time: 2024-04-18T07:52:31Z
	// distance: 1.7
	// location: 48.8737820, 2.2950183

	// Save to a file called "end.pb", then load from it.

	t := time.Date(2024, time.April, 18, 7, 52, 31, 0, time.UTC)
	msg := pb.RideEnd{
		Id:       "aa08deb",
		Time:     timestamppb.New(t),
		Distance: 1.7,
		Location: &pb.Location{
			Lat: 48.8737820,
			Lng: 2.2950183,
		},
	}
	fmt.Println("msg :", &msg)

	data, err := proto.Marshal(&msg)
	if err != nil {
		fmt.Println("ERROR:", err)
		return
	}

	const fileName = "end.pb"
	file, err := os.Create(fileName)
	if err != nil {
		fmt.Println("ERROR:", err)
		return
	}
	defer file.Close()

	if _, err := io.Copy(file, bytes.NewReader(data)); err != nil {
		fmt.Println("ERROR:", err)
		return
	}

	file, err = os.Open(fileName)
	if err != nil {
		fmt.Println("ERROR:", err)
		return
	}
	defer file.Close()

	data, err = io.ReadAll(file)
	if err != nil {
		fmt.Println("ERROR:", err)
		return
	}

	var msg2 pb.RideEnd
	if err := proto.Unmarshal(data, &msg2); err != nil {
		fmt.Println("ERROR:", err)
		return
	}

	fmt.Println("msg2:", &msg2)
}
```

## Other serialization formats

### YAML and TOML

```yaml
---
version: 1

server:
  port: 8080
logging:
  level: INFO
```

```go
package main

import (
	"fmt"
	"os"

	"gopkg.in/yaml.v3"
)

type Config struct {
	Version int
	Server  struct {
		Port int
	}
	Logging struct {
		Level string
	}
}

func main() {
	file, err := os.Open("config.yml")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}
	defer file.Close()

	var cfg Config
	dec := yaml.NewDecoder(file)
	if err := dec.Decode(&cfg); err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}

	fmt.Printf("%+v\n", cfg)
}
```

[Go toml](https://pkg.go.dev/github.com/BurntSushi/toml)
[Go viper](https://pkg.go.dev/github.com/spf13/viper)

### XML

[Go XML](https://pkg.go.dev/encoding/xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<store>
    <item sku="m183x">
	<name>Magic Wand</name>
	<price>7.0</price>
    </item>
    <item sku="m184y">
	<name>Invisibility Cape</name>
	<price>13.2</price>
    </item>
    <item sku="m185z">
	<name>Levitation Spell</name>
	<price>9.3</price>
    </item>
</store>
```

```go
package main

import (
	"encoding/xml"
	"fmt"
	"os"
)

type Store struct {
	Items []struct {
		SKU   string  `xml:"sku,attr"`
		Name  string  `xml:"name"`
		Price float64 `xml:"price"`
	} `xml:"item"`
}

func main() {
	file, err := os.Open("store.xml")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}
	defer file.Close()

	var s Store
	dec := xml.NewDecoder(file)
	if err := dec.Decode(&s); err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}

	fmt.Printf("%+v\n", s)
}
```

### CSV

[Go cvs](https://pkg.go.dev/encoding/csv)
[Go csvutil](https://pkg.go.dev/github.com/jszwec/csvutil)

```csv
sku,name,price (galleons)
m183x,Magic Wand,7.0
m184y,Invisibility Cape,13.2
m185z,Levitation Spell,9.3
```

```go
package main

import (
	"encoding/csv"
	"errors"
	"fmt"
	"io"
	"os"
)

func main() {
	file, err := os.Open("store.csv")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}
	defer file.Close()

	r := csv.NewReader(file)
	for {
		fields, err := r.Read()
		if errors.Is(err, io.EOF) {
			break
		}
		if err != nil {
			fmt.Fprintf(os.Stderr, "error: %s\n", err)
			os.Exit(1)
		}

		fmt.Println(fields)
	}
}
```

### SQL

[SQLite](https://pkg.go.dev/github.com/mattn/go-sqlite3)

```go
package main

import (
	"database/sql"
	_ "embed"
	"fmt"
	"os"

	_ "modernc.org/sqlite"
)

type Item struct {
	SKU   string
	Name  string
	Price float64
}

var store = []Item{
	{"m183x", "Magic Wand", 7.0},
	{"m184y", "Invisibility Cape", 13.2},
	{"m185z", "Levitation Spell", 9.3},
}

var (
	//go:embed insert.sql
	insertSQL string

	//go:embed get.sql
	getSQL string
)

func main() {
	db, err := sql.Open("sqlite", "store.db")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}
	defer db.Close()

	// Save
	for _, item := range store {
		if _, err := db.Exec(insertSQL, item.SKU, item.Name, item.Price); err != nil {
			fmt.Fprintf(os.Stderr, "error: %s\n", err)
			os.Exit(1)
		}
	}

	// Load
	const sku = "m184y"
	row := db.QueryRow(getSQL, sku)
	if row.Err(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}

	i := Item{
		SKU: sku,
	}
	if err := row.Scan(&i.Name, &i.Price); err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}

	fmt.Printf("%+v\n", i)
}
```

### Challenge ETL

```go
package main

import (
	"database/sql"
	_ "embed"
	"encoding/xml"
	"fmt"
	"io"
	"os"
	"path/filepath"
	"time"

	_ "modernc.org/sqlite"
)

//go:embed insert.sql
var insertSQL string

func insert(db *sql.DB, r io.Reader) (int, error) {
	var doc struct {
		Logs []struct {
			Time    time.Time `xml:"time"`
			Level   string    `xml:"level"`
			Message string    `xml:"message"`
		} `xml:"log"`
	}

	dec := xml.NewDecoder(r)
	if err := dec.Decode(&doc); err != nil {
		return 0, err
	}

	for _, log := range doc.Logs {
		if _, err := db.Exec(insertSQL, log.Time, log.Level, log.Message); err != nil {
			return 0, err
		}
	}

	return len(doc.Logs), nil
}

func main() {
	matches, err := filepath.Glob("logs/*.xml")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}

	db, err := sql.Open("sqlite", "logs.db")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %s\n", err)
		os.Exit(1)
	}
	defer db.Close()

	for _, fileName := range matches {
		file, err := os.Open(fileName)
		if err != nil {
			fmt.Fprintf(os.Stderr, "error: %s\n", err)
			os.Exit(1)
		}
		defer file.Close()

		n, err := insert(db, file)
		if err != nil {
			fmt.Fprintf(os.Stderr, "error: %s\n", err)
			os.Exit(1)
		}
		fmt.Printf("%s: %d records\n", fileName, n)
	}
}
```
