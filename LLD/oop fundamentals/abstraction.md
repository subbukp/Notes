# OOP — Abstraction

## The Core Idea

> Abstraction = Hiding Complexity + Showing Essentials

Separate **what** an object does from **how** it does it.

---

## 1. Real-World Analogy — Driving a Car

```mermaid
flowchart LR
    Driver -->|steer| SW[Steering Wheel]
    Driver -->|accelerate| Pedal[Accelerator]
    Driver -->|shift| Gear[Gear Lever]

    SW -.->|hidden| E1[Transmission]
    Pedal -.->|hidden| E2[Fuel Injection]
    Gear -.->|hidden| E3[Torque / Combustion]

    style E1 fill:#888,color:#fff
    style E2 fill:#888,color:#fff
    style E3 fill:#888,color:#fff
```

You interact with simple controls. The mechanical complexity is hidden. That's abstraction.

---

## 2. Why Abstraction Matters

### Without Abstraction — Tightly Coupled

```mermaid
flowchart TD
    LS[LoggingService] -->|if type == console| C[print to stdout]
    LS -->|if type == file| F[write to file]
    LS -->|if type == remote| R[HTTP POST]
    LS -->|if type == database| D[INSERT query]

    style LS fill:#C00000,color:#fff
```

Every new destination = another branch in `LoggingService`. Testing one logger requires the whole class. Changing one format risks breaking others.

### With Abstraction — Decoupled

```mermaid
flowchart TD
    App[Application] -->|logger.log| Abstract["«abstract»\nLogger"]
    Abstract -->|implemented by| CL[ConsoleLogger]
    Abstract -->|implemented by| FL[FileLogger]
    Abstract -->|implemented by| RL[RemoteLogger]
    Abstract -->|implemented by| DL[DatabaseLogger]

    style Abstract fill:#1F4E79,color:#fff
```

`Application` only knows about `Logger`. Swap implementations by changing one line.

---

## 3. How Abstraction Is Achieved

```mermaid
flowchart TD
    A[Abstraction] --> B[Abstract Classes]
    A --> C[Interfaces]
    A --> D[Clean Public APIs]

    B --> B1[Shared behaviour\n+ contract]
    C --> C1[Contract only\nno shared behaviour]
    D --> D1[Hide internals behind\npublic methods]
```

---

## 4. Abstract Classes

```mermaid
classDiagram
    class Logger {
        <<abstract>>
        #level: str
        +log(message: str)*
        +format_message(message: str) str
    }

    class ConsoleLogger {
        +log(message: str)
    }

    class FileLogger {
        -file_path: str
        +log(message: str)
    }

    Logger <|-- ConsoleLogger
    Logger <|-- FileLogger
```

```python
from abc import ABC, abstractmethod
from datetime import datetime

class Logger(ABC):
    def __init__(self, level: str):
        self.level = level

    # concrete — shared by all subclasses
    def format_message(self, message: str) -> str:
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        return f"[{timestamp}] [{self.level}] {message}"

    # abstract — each subclass must implement
    @abstractmethod
    def log(self, message: str) -> None:
        pass


class ConsoleLogger(Logger):
    def log(self, message: str) -> None:
        print(self.format_message(message))    # inherits format_message for free


class FileLogger(Logger):
    def __init__(self, level: str, file_path: str):
        super().__init__(level)
        self.file_path = file_path

    def log(self, message: str) -> None:
        with open(self.file_path, "a") as f:
            f.write(self.format_message(message) + "\n")
```

> **Key value over interfaces** — `format_message()` is written once and inherited. Without abstraction you'd duplicate that logic in every logger.

---

## 5. Interfaces as Abstraction

When **unrelated classes** share a capability (not a family relationship), use an interface.

```mermaid
classDiagram
    class Exportable {
        <<interface>>
        +export() str
    }

    class CSVExporter {
        +export() str
    }

    class JSONExporter {
        +export() str
    }

    class XMLExporter {
        +export() str
    }

    Exportable <|.. CSVExporter
    Exportable <|.. JSONExporter
    Exportable <|.. XMLExporter
```

```python
from abc import ABC, abstractmethod

class Exportable(ABC):
    @abstractmethod
    def export(self) -> str:
        pass

class CSVExporter(Exportable):
    def export(self) -> str:
        return "id,name,value\n1,scan,0.91"

class JSONExporter(Exportable):
    def export(self) -> str:
        return '{"id": 1, "name": "scan", "value": 0.91}'
```

No shared behaviour — purely a contract. Any code that needs to export depends on `Exportable`, not on a specific exporter.

---

## 6. Public APIs as Abstraction

No inheritance needed. A well-designed class that hides internal complexity behind clean public methods **is** abstraction.

```mermaid
sequenceDiagram
    participant Caller
    participant DatabaseClient

    Caller->>DatabaseClient: connect()
    DatabaseClient->>DatabaseClient: __create_pool()
    DatabaseClient->>DatabaseClient: __authenticate()

    Caller->>DatabaseClient: query("SELECT ...")
    DatabaseClient->>DatabaseClient: __parse_query()
    DatabaseClient->>DatabaseClient: __retry_logic()
    DatabaseClient-->>Caller: results
```

```python
class DatabaseClient:
    def connect(self, host: str, port: int) -> None:
        self.__create_pool(host, port)   # hidden
        self.__authenticate()            # hidden

    def query(self, sql: str) -> list:
        parsed = self.__parse_query(sql) # hidden
        return self.__execute(parsed)    # hidden

    # private — caller never sees these
    def __create_pool(self, host, port): ...
    def __authenticate(self): ...
    def __parse_query(self, sql): ...
    def __execute(self, parsed): ...


# Caller sees only this
client = DatabaseClient()
client.connect("db.hospital.com", 5432)
results = client.query("SELECT * FROM patients")
```

---

## 7. Abstraction vs Encapsulation

```mermaid
flowchart LR
    subgraph Abstraction["Abstraction — external view"]
        A1[What does it do?]
        A2[Simplify usage]
        A3[Design-level]
        A4["accelerate() hides\nfuel injection logic"]
    end

    subgraph Encapsulation["Encapsulation — internal view"]
        E1[How is data protected?]
        E2[Restrict access to state]
        E3[Implementation-level]
        E4["__balance hidden\nbehind deposit()"]
    end

    Abstraction <-->|"work together"| Encapsulation
```

| Aspect | Encapsulation | Abstraction |
|---|---|---|
| Focus | Protecting data | Hiding complexity |
| Goal | Restrict access to state | Simplify usage |
| Level | Implementation | Design |
| Example | `__balance` in `BankAccount` | Exposing only `deposit()` / `withdraw()` |

> **Encapsulation protects. Abstraction simplifies.**

---

## 8. Practical Example — Media Player

```mermaid
classDiagram
    class MediaPlayer {
        <<abstract>>
        #player_name: str
        +play()*
        +pause()*
        +stop()*
        +display_status()
        +log_action(action: str)
    }

    class AudioPlayer {
        -audio_file: str
        +play()
        +pause()
        +stop()
    }

    class VideoPlayer {
        -video_file: str
        -resolution: str
        +play()
        +pause()
        +stop()
    }

    class StreamingPlayer {
        -stream_url: str
        -buffer_size: int
        +play()
        +pause()
        +stop()
    }

    class PlayerController {
        -player: MediaPlayer
        +start_playback()
        +pause_playback()
        +stop_playback()
    }

    MediaPlayer <|-- AudioPlayer
    MediaPlayer <|-- VideoPlayer
    MediaPlayer <|-- StreamingPlayer
    PlayerController --> MediaPlayer
```

### Lifecycle Flow

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Playing : start_playback()
    Playing --> Paused : pause_playback()
    Paused --> Playing : start_playback()
    Playing --> Stopped : stop_playback()
    Paused --> Stopped : stop_playback()
    Stopped --> [*]
```

```python
from abc import ABC, abstractmethod

class MediaPlayer(ABC):
    def __init__(self, player_name: str):
        self.player_name = player_name

    @abstractmethod
    def play(self) -> None: pass

    @abstractmethod
    def pause(self) -> None: pass

    @abstractmethod
    def stop(self) -> None: pass

    # concrete — inherited by all players
    def display_status(self) -> None:
        print(f"[{self.player_name}] ready.")

    def log_action(self, action: str) -> None:
        print(f"[LOG] {self.player_name}: {action}")


class AudioPlayer(MediaPlayer):
    def __init__(self, audio_file: str):
        super().__init__("AudioPlayer")
        self.audio_file = audio_file

    def play(self)  -> None: self.log_action(f"Playing audio: {self.audio_file}")
    def pause(self) -> None: self.log_action("Audio paused")
    def stop(self)  -> None: self.log_action("Audio stopped")


class StreamingPlayer(MediaPlayer):
    def __init__(self, stream_url: str, buffer_size: int):
        super().__init__("StreamingPlayer")
        self.stream_url = stream_url
        self.buffer_size = buffer_size

    def play(self)  -> None: self.log_action(f"Buffering {self.buffer_size}KB → streaming {self.stream_url}")
    def pause(self) -> None: self.log_action("Stream paused")
    def stop(self)  -> None: self.log_action("Stream stopped, buffer cleared")


class PlayerController:
    def __init__(self, player: MediaPlayer):
        self.player = player           # depends on abstraction, not concrete class

    def start_playback(self)  -> None: self.player.play()
    def pause_playback(self)  -> None: self.player.pause()
    def stop_playback(self)   -> None: self.player.stop()


# Swap player — controller unchanged
controller = PlayerController(AudioPlayer("scan_report.mp3"))
controller.start_playback()

controller = PlayerController(StreamingPlayer("https://stream.hospital.com/live", 512))
controller.start_playback()
```

---

## Quick Reference

| Mechanism | Has shared behaviour | Has contract | Use when |
|---|---|---|---|
| Abstract class | ✅ | ✅ | Related classes sharing common logic |
| Interface | ❌ | ✅ | Unrelated classes sharing a capability |
| Public API | ✅ | ❌ | Single class hiding internal complexity |

---

## Summary — All Five OOP Concepts

```mermaid
mindmap
  root((OOP))
    Classes & Objects
      Class = blueprint
      Object = instance
      Independent state per object
    Enums
      Fixed set of named constants
      str Enum for serialization
      int Enum for ordering
      MRO = left to right lookup
    Interfaces
      Contract only — no implementation
      Enables polymorphism
      Dependency injection
      Program to the interface
    Encapsulation
      Data hiding via private fields
      Controlled access via methods
      Validation at the boundary
      Protects internal state
    Abstraction
      Hides complexity
      Shows only essentials
      Abstract class = contract + shared logic
      Interface = contract only
      Public API = clean surface
```

> **Classes** give structure. **Enums** give safe constants. **Interfaces** give contracts. **Encapsulation** protects state. **Abstraction** simplifies complexity.