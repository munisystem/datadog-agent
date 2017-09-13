## package `tagger`

The **Tagger** is the central source of truth for client-side entity tagging. It
runs **Collector**s that detect entities and collect their tags. Tags are then
stored in memory (by the **TagStore**) and can be queried by the Tagger.Tag()
method. Calling once Tagger.Init() after the **config** package is ready is
needed to enable collection.

The package methods use a common **DefaultTagger** object, but we can create
a custom **Tagger** object for testing.

The package will implement an IPC mechanism (a server and a client) to allow
other agents to query the **DefaultTagger** and avoid duplicating the information
in their process. Switch between local and client mode will be done via a build flag.

### Collector
A **Collector** connects to a single information source and pushes **TagInfo**
structs to a channel, towards the **Tagger**. It can either run in streaming
mode or pull mode, depending of what's most efficient for the data source:

#### Streamer
The **DockerCollector** runs in stream mode as it collects events from the docker
daemon and reacts to them, sending updates incrementally.

#### Puller
The **KubernetesCollector** and **ECSCollector** will run in pull mode as they
need to query and filter a full entity list every time. They will only push
updates to the store though, by keeping an internal state of the latest
revision.

### TagStore
The **TagStore** reads **TagInfo** structs and stores them in a in-memory
cache. Cache invalidation is triggered by the collectors (or source) by either:

  - sending new tags for the same `Entity`, all the tags from this `Source`
  will be removed and replaced by the new tags
  - sending a **TagInfo** with **DeleteEntity** set, all the entries for this
  entity (including from other sources) will be deleted

### Tagger
The Tagger handles the glue between **Collectors** and **TagStore** and the
cache miss logic. If the tags from the **TagStore** are missing some sources,
they will be manually queried in a block way, and the cache will be updated.

For convenience, the package creates a **DefaultTagger** object that is used
when calling the `tagger.Tag()` method.


                   +-----------+
                   | Collector |
                   +---+-------+
                       |
                       |
    +--------+      +--+-------+       +-------------+
    |  User  <------+  Tagger  +-------> IPC handler |
    |packages|      +--+-----^-+       +-------------+
    +--------+         |     |
                       |     |
                    +--v-----+-+
                    | TagStore |
                    +----------+