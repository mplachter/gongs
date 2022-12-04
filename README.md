# GONGQ - Go Nats Generic Queues

A thin wrapper around Nats Jetstream client that uses Go Generics to provide strongly typed NATS work queues.

## Example Usage

Define the Type that will be published and retrieved from the NATS queue:

```go
type ExampleMsgEventData struct {
	Id          string
	Type        string
	Description string
}

type ExampleMsg struct {
	eventData *ExampleMsgEventData
}

// Mandatory - Implement the `gongq.MsgEvent` interface
func (e *ExampleMsg) GetId() string {
	return e.eventData.Id
}

func (e *ExampleMsg) DecodeEventData(b []byte) error {
	d := &ExampleMsgEventData{}
	json.Unmarshal(b, d)
	e.eventData = d
	return nil
}

func (e *ExampleMsg) EncodeEventData() []byte {
	b, _ := json.Marshal(e.eventData)
	return b
}
```

Create Generic Queue for the above type:

```go
	// create Jetstream for Queue
	cfg := &nats.StreamConfig{
		Name:      "EXAMPLE",
		Subjects:  []string{"example.>"},
		Storage:   nats.MemoryStorage,
		Retention: nats.WorkQueuePolicy,
	}
	js, _ := nc.JetStream()
	js.AddStream(cfg)

	// create Generic Queue
	q := gongq.NewGenericQueue[ExampleMsg](js, "example.events", cfg.Name)
```

Publish event

```go
	// Publish an event
	q.Publish(&ExampleMsg{
		eventData: &ExampleMsgEventData{
			Id:          "abc123",
			Type:        "start",
			Description: "An important task has started",
		},
	})
```

Read last event off queue

```go
	// Read event from NATS
	event, _ := q.GetLastMsg("example")

	fmt.Printf("Id: %s [%s] - %s",
		event.eventData.Id,
		event.eventData.Type,
		event.eventData.Description,
	)
```