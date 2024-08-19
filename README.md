# Playlist Service - System Design & Implementation

## Overview

This project implements a playlist service that maintains a 24/7 queue of main events with inner events nested within them. The service ensures that main events always play consecutively without gaps, recalculates start times dynamically, and allows efficient user interaction via a React-based frontend. The backend is built with Express.js, TypeScript, and PostgreSQL, handling all the business logic for event management.

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li><a href="#architecture">Architecture</a></li>
    <li><a href="#key-features">Key Features</a></li>
    <li><a href="#data-models-and-database-schema">Data Models and Database Schema</a></li>
    <li><a href="#api-routes-implementation">API Routes Implementation</a></li>
    <li><a href="#handling-key-challenges">Handling Key Challenges</a></li>
    <li><a href="#optimization-techniques">Optimization Techniques</a></li>
    <li><a href="#real-time-updates">Real-Time Updates</a></li>
    <li><a href="#frontend-implementation">Frontend Implementation</a></li>
    <li><a href="#setup-instructions">Setup Instructions</a></li>
    <li><a href="#backend-system-optimizations">Backend System Optimizations</a></li>
    <li><a href="#frontend-user-experience-optimizations">Frontend User Experience Optimizations</a></li>
    <li><a href="#summary-of-optimizations">Summary of Optimizations</a></li>
  </ol>
</details>

## Architecture

### Backend: Express.js (TypeScript)

- API Services: Handles CRUD operations for main and inner events.
- PostgreSQL Database: Stores events with structured schemas.
- Real-Time Updates: Uses Socket.IO for real-time event synchronization.
- Transaction Handling: Ensures atomicity in operations like event addition, updates, and deletions.

### Frontend: React (TypeScript)

- User Interface: Displays events, with infinite scrolling and smooth interactions.
- Real-Time Updates: Integrates with WebSocket to receive event changes without refreshing.

## Key Features

### Event Management:

- Add, update, delete main and inner events.
- Recalculate start times for subsequent events after modification.
- Validate inner events to ensure they fit within parent main event durations.

### Real-Time Synchronization:

- Ensure users see the latest playlist updates through WebSockets.

### Efficient Pagination:

- Use efficient pagination and batch updates to handle large event lists.

## Data Models and Database Schema

### Main Event Model (TypeScript)

```ts
export interface MainEvent {
  id: string;
  title: string;
  absoluteStartTime: Date;
  duration: number; // in milliseconds
  status: "completed" | "scheduled" | "in-progress" | "deleted";
}
```

### Inner Event Model (TypeScript)

```ts
export interface InnerEvent {
  id: string;
  mainEventId: string;
  title: string;
  relativeStartTime: number; // in milliseconds relative to main event's start
  duration: number;
  status: "completed" | "scheduled" | "in-progress" | "deleted";
}
```

### SQL Database Schema

```sql
CREATE TABLE main_events (
  id UUID PRIMARY KEY,
  title VARCHAR(255),
  absolute_start_time TIMESTAMP NOT NULL,
  duration INT NOT NULL,
  status VARCHAR(50) CHECK (status IN ('completed', 'scheduled', 'in-progress', 'deleted')) NOT NULL
);

CREATE INDEX idx_absolute_start_time ON main_events (absolute_start_time);

CREATE TABLE inner_events (
  id UUID PRIMARY KEY,
  main_event_id UUID REFERENCES main_events(id) ON DELETE CASCADE,
  title VARCHAR(255),
  relative_start_time INT NOT NULL,
  duration INT NOT NULL,
  status VARCHAR(50) CHECK (status IN ('completed', 'scheduled', 'in-progress', 'deleted')) NOT NULL
);

CREATE INDEX idx_main_event_id ON inner_events (main_event_id);
```

- `main_events` Table:
  - Contains the details of each main event, including `absolute_start_time`, `duration`, and `status`.
  - Indexed on `absolute_start_time` for efficient retrieval of upcoming events.
- `inner_events` Table:
  - Stores inner events linked to a main event via `main_event_id`.
  - Indexed on `main_event_id` for efficient querying of inner events by their parent event.

## API Routes Implementation

### Add Main Event

- Endpoint: `POST /main-events`
- Description: Creates a new main event and recalculates start times for all subsequent events.

```ts
import express, { Request, Response } from "express";
import { getRepository, getManager } from "typeorm";
import { MainEvent } from "../entities/MainEvent";

const router = express.Router();

router.post("/main-events", async (req: Request, res: Response) => {
  const { title, duration } = req.body;
  const mainEventRepository = getRepository(MainEvent);

  const newEvent = mainEventRepository.create({
    title,
    duration,
    status: "scheduled",
    absoluteStartTime: new Date(), // placeholder, recalculate based on last event
  });

  try {
    await getManager().transaction(async (transactionalEntityManager) => {
      await transactionalEntityManager.save(newEvent);
      await transactionalEntityManager.query(
        "SELECT recalculate_main_event_times($1)",
        [newEvent.id]
      );
    });
    res.status(201).json(newEvent);
  } catch (error) {
    res.status(500).json({ message: "Error creating event", error });
  }
});

export default router;
```

### Update Main Event Duration

- Endpoint: `PUT /main-events/:id`
- Description: Updates the duration of a main event and recalculates start times of subsequent events.

```ts
router.put("/main-events/:id", async (req: Request, res: Response) => {
  const { id } = req.params;
  const { duration } = req.body;
  const mainEventRepository = getRepository(MainEvent);

  try {
    await getManager().transaction(async (transactionalEntityManager) => {
      await transactionalEntityManager.update(MainEvent, id, { duration });
      await transactionalEntityManager.query(
        "SELECT recalculate_main_event_times($1)",
        [id]
      );
    });
    res.status(200).json({ message: "Event updated successfully" });
  } catch (error) {
    res.status(500).json({ message: "Error updating event", error });
  }
});
```

### Delete Main Event

- Endpoint: `DELETE /main-events/:id`
- Description: Deletes a main event and recalculates start times of subsequent events.

```ts
router.delete("/main-events/:id", async (req: Request, res: Response) => {
  const { id } = req.params;

  try {
    await getManager().transaction(async (transactionalEntityManager) => {
      await transactionalEntityManager.delete(MainEvent, id);
      await transactionalEntityManager.query(
        "SELECT recalculate_main_event_times($1)",
        [id]
      );
    });
    res.status(200).json({ message: "Event deleted successfully" });
  } catch (error) {
    res.status(500).json({ message: "Error deleting event", error });
  }
});
```

### Add Inner Event

- Endpoint: `POST /inner-events`
- Description: Adds an inner event to a specific main event.

```ts
router.post("/inner-events", async (req: Request, res: Response) => {
  const { mainEventId, title, relativeStartTime, duration } = req.body;
  const innerEventRepository = getRepository(InnerEvent);

  const newInnerEvent = innerEventRepository.create({
    mainEventId,
    title,
    relativeStartTime,
    duration,
    status: "scheduled",
  });

  try {
    await getManager().save(newInnerEvent);
    res.status(201).json(newInnerEvent);
  } catch (error) {
    res.status(500).json({ message: "Error creating inner event", error });
  }
});
```

## Handling Key Challenges

### Recalculating Start Times

We use a stored procedure in PostgreSQL to batch update subsequent events when a main event is added, updated, or deleted.

```sql
CREATE OR REPLACE FUNCTION recalculate_main_event_times(start_event_id UUID)
RETURNS VOID AS $$
DECLARE
    event RECORD;
    previous_event RECORD;
    start_time TIMESTAMP;
BEGIN
    SELECT absolute_start_time INTO start_time FROM main_events WHERE id = start_event_id;

    FOR event IN
        SELECT * FROM main_events WHERE absolute_start_time > start_time ORDER BY absolute_start_time ASC
    LOOP
        UPDATE main_events
        SET absolute_start_time = previous_event.absolute_start_time + INTERVAL '1 second' * previous_event.duration
        WHERE id = event.id;

        SELECT * INTO previous_event FROM main_events WHERE id = event.id;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### Handling Race Conditions

We use row-level locking with pessimistic_write to avoid race conditions when updating events.

```ts
async function updateMainEventWithLock(eventId: string, newDuration: number) {
  await getManager().transaction(async (transactionalEntityManager) => {
    const mainEvent = await transactionalEntityManager
      .createQueryBuilder(MainEvent, "main_event")
      .setLock("pessimistic_write")
      .where("main_event.id = :id", { id: eventId })
      .getOne();

    if (mainEvent) {
      mainEvent.duration = newDuration;
      await transactionalEntityManager.save(mainEvent);
      await transactionalEntityManager.query(
        "SELECT recalculate_main_event_times($1)",
        [mainEvent.id]
      );
    }
  });
}
```

## Optimization Techniques

### Batch Updates

To ensure efficiency, we update event start times in batches rather than individually. This reduces the number of database roundtrips, which is critical for large event lists.

TypeScript Code for Batch Update:

```ts
async function batchUpdateStartTimes(events: MainEvent[]) {
  const queryRunner = getManager().queryRunner;
  await queryRunner.startTransaction();

  try {
    const updatePromises = events.map((event) => {
      return queryRunner.manager.update(MainEvent, event.id, {
        absoluteStartTime: event.absoluteStartTime,
      });
    });
    await Promise.all(updatePromises);

    await queryRunner.commitTransaction();
  } catch (error) {
    await queryRunner.rollbackTransaction();
    throw error;
  } finally {
    await queryRunner.release();
  }
}
```

In this example, we're leveraging TypeORM's `queryRunner` to handle the database transaction, ensuring that all updates are applied in a single batch. This minimizes the overhead of issuing individual queries for each event, making the system more efficient when dealing with large data sets.

### Efficient Pagination

Efficient pagination ensures that the frontend only fetches the data it needs, especially when dealing with a large number of events. The server should implement pagination using `OFFSET` and `LIMIT`.

TypeScript Backend Code for Paginated Queries:

```ts
async function getPaginatedMainEvents(
  offset: number,
  limit: number
): Promise<MainEvent[]> {
  const mainEventRepository = getRepository(MainEvent);
  return await mainEventRepository
    .createQueryBuilder("main_event")
    .orderBy("main_event.absolute_start_time", "ASC")
    .offset(offset)
    .limit(limit)
    .getMany();
}
```

This paginated query allows the client to fetch events in chunks (e.g., 20 events at a time) and supports smooth infinite scrolling on the frontend.

### Efficient Query Optimization Using Indexes

We ensure that frequently queried columns such as `absolute_start_time` (in the `main_events` table) and `main_event_id` (in the `inner_events` table) are indexed. These indexes speed up queries and significantly reduce the time needed to fetch large data sets.

```sql
CREATE INDEX idx_absolute_start_time ON main_events (absolute_start_time);
CREATE INDEX idx_main_event_id ON inner_events (main_event_id);
```

These indexes allow the database to efficiently locate events based on their start times and quickly find all inner events associated with a specific main event.

## Real-Time Updates

We use WebSockets to notify the frontend in real-time when main or inner events are modified (added, updated, deleted). This ensures that users always see the latest data without having to refresh the page.

### WebSocket Backend Implementation (Express + Socket.IO)

Install Dependencies:

```bash
npm install socket.io
```

Backend WebSocket Code:

```ts
import { Server } from "socket.io";

const io = new Server(httpServer, {
  cors: {
    origin: "*", // adjust CORS policy as needed
  },
});

io.on("connection", (socket) => {
  console.log("New client connected");

  // Broadcast update to all clients when an event is updated
  socket.on("event-updated", (data) => {
    io.emit("update-playlist", data); // send update to all connected clients
  });

  socket.on("disconnect", () => {
    console.log("Client disconnected");
  });
});

httpServer.listen(3000, () => {
  console.log("Server is running on port 3000");
});
```

This Socket.IO setup broadcasts any updates to all connected clients, ensuring that the playlist view in the browser remains consistent across multiple users.

### WebSocket Frontend (React)

The frontend listens for real-time updates via WebSocket and updates the playlist accordingly.

Frontend WebSocket Code:

```ts
import io from "socket.io-client";
import { useEffect, useState } from "react";

const socket = io("http://localhost:3000");

function Playlist() {
  const [events, setEvents] = useState([]);

  useEffect(() => {
    // Listen for playlist updates
    socket.on("update-playlist", (data) => {
      // Update the playlist with new data
      setEvents((prevEvents) => [...prevEvents, ...data]);
    });

    return () => {
      socket.disconnect();
    };
  }, []);

  return <div>{/* Render events */}</div>;
}
```

Whenever the backend broadcasts an event update, this WebSocket client receives the data and updates the state of the `events` array in React. This way, users always have the latest view of the playlist.

## Frontend Implementation

The frontend of this system is designed to handle large event lists efficiently. It includes infinite scrolling and real-time updates via WebSockets.

### Infinite Scrolling in React

As users scroll down the list of events, the frontend will dynamically load more events from the server. This allows the UI to handle large datasets without slowing down or consuming too much memory.

Frontend Infinite Scrolling Code:

```ts
import { useEffect, useState } from "react";
import axios from "axios";

function Playlist() {
  const [events, setEvents] = useState([]);
  const [offset, setOffset] = useState(0);
  const limit = 20;

  useEffect(() => {
    const loadEvents = async () => {
      const response = await axios.get(
        `/api/main-events?offset=${offset}&limit=${limit}`
      );
      setEvents((prevEvents) => [...prevEvents, ...response.data]);
    };

    loadEvents();
  }, [offset]);

  const handleScroll = () => {
    if (window.innerHeight + window.scrollY >= document.body.offsetHeight) {
      setOffset(offset + limit); // Load more events when reaching the bottom of the page
    }
  };

  useEffect(() => {
    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, []);

  return <div>{/* Render events */}</div>;
}
```

The `handleScroll` function listens for the user scrolling to the bottom of the page and then increments the `offset` to load more events from the server. The events are then appended to the current state of the event list.

## Setup Instructions

### Backend Setup

- Clone the repository:
  ```bash
  git clone https://github.com/akashungarala/playlist-service.git
  cd backend
  ```
- Install dependencies:
  ```bash
  npm install
  ```
- Setup PostgreSQL:
  - Create a PostgreSQL database and configure your connection settings in `.env`.
- Run Migrations:
  ```bash
  npm run migration:run
  ```
- Start the server:
  ```bash
  npm run start
  ```

### Frontend Setup

- Navigate to the frontend directory:
  ```bash
  cd frontend
  ```
- Install dependencies:
  ```bash
  npm install
  ```
- Start the React app:
  ```bash
  npm start
  ```

## Opportunities to Enhance System Design

While this system design is quite efficient, there are still some additional optimizations we can apply to further enhance backend performance and real-time frontend user experience. Here are a few advanced strategies and techniques that can be implemented:

## Backend System Optimizations

### Query Optimization with Caching

- Challenge: Even with optimized SQL queries and indexing, repeatedly fetching similar data (e.g., the current and next 10 events) from the database can cause delays.
- Solution: Use in-memory caching (e.g., Redis) to cache frequently accessed data, such as upcoming events.
  - Benefit: Reduces database load by avoiding repeated queries for commonly accessed data.
- Example Implementation: Cache the current and upcoming events for fast retrieval.

  ```ts
  import Redis from "ioredis";
  const redis = new Redis();

  async function getCurrentAndUpcomingEvents() {
    const cachedEvents = await redis.get("upcoming-events");
    if (cachedEvents) {
      return JSON.parse(cachedEvents);
    }

    const events = await getRepository(MainEvent).find({
      where: { status: "scheduled" },
      order: { absoluteStartTime: "ASC" },
      take: 10,
    });

    await redis.set("upcoming-events", JSON.stringify(events), "EX", 60); // Cache for 60 seconds
    return events;
  }
  ```

### Database Partitioning and Sharding

- Challenge: As the database grows with many events, querying large tables can still be a bottleneck.
- Solution: Partition the `main_events` and `inner_events` tables by time or other criteria to distribute the data more efficiently. Sharding can be used for horizontal scaling.
  - Benefit: Improves query performance by limiting the size of tables queried during each operation.
- Example: Partition `main_events` by year or month, so that queries only scan relevant partitions.

### Background Jobs for Long-Running Processes

- Challenge: Recalculating start times for a large number of events in real-time could lead to delayed responses or blocking requests.
- Solution: Offload these tasks to a background job queue (e.g., Bull or Kue for Node.js) that recalculates start times asynchronously.
  - Benefit: Allows the API to respond immediately, offloading computationally heavy operations to background workers.
- Example Implementation:

  ```ts
  import Queue from "bull";
  const recalculateQueue = new Queue("recalculate");

  // Add the recalculation task to the queue
  recalculateQueue.add({ eventId });

  recalculateQueue.process(async (job) => {
    const { eventId } = job.data;
    await getManager().query("SELECT recalculate_main_event_times($1)", [
      eventId,
    ]);
  });
  ```

### Eventual Consistency for Non-Critical Updates

- Challenge: Ensuring strict consistency with every operation can lead to increased response times, especially for large datasets.
- Solution: For non-critical updates (e.g., inner event duration changes), consider applying eventual consistency where updates are propagated to the system asynchronously.
  - Benefit: Improves performance by allowing the system to operate without waiting for all updates to complete before returning a response.

### Efficient Bulk Inserts/Updates

- Challenge: Performing individual inserts/updates for a large batch of events can be inefficient.
- Solution: Use bulk insert/update queries with optimized SQL statements to insert or update multiple rows in a single database operation.
- Example: PostgreSQL supports bulk inserts like this:
  ```sql
  INSERT INTO main_events (id, title, absolute_start_time, duration, status)
  VALUES
  ('id1', 'Event 1', '2024-01-01 10:00', 60000, 'scheduled'),
  ('id2', 'Event 2', '2024-01-01 11:00', 120000, 'scheduled')
  ON CONFLICT (id) DO UPDATE
  SET absolute_start_time = EXCLUDED.absolute_start_time, duration = EXCLUDED.duration;
  ```

## Frontend User Experience Optimizations

### Optimized WebSocket Communication

- Challenge: Sending unnecessary WebSocket updates for every minor change can overwhelm the client, especially with large data sets.
- Solution: Use delta updates instead of sending the full event list. Only send the changes that occurred (e.g., a specific event was added, updated, or deleted).
  - Benefit: Reduces the data transmitted over WebSockets, leading to faster updates and less strain on the frontend.
- Example Implementation:

  ```ts
  socket.on("update-playlist", (delta) => {
    setEvents((prevEvents) => {
      // Apply delta changes (additions, updates, deletions)
      return applyDelta(prevEvents, delta);
    });
  });
  ```

### Throttling and Debouncing Scroll Events

- Challenge: Handling too many `scroll` events in the frontend can degrade performance, especially with infinite scrolling.
- Solution: Throttle or debounce the scroll event to prevent it from firing too frequently.

  - Example: Use a debounce or throttle function to optimize scroll handling in React.

  ```ts
  import { throttle } from "lodash";

  const handleScroll = throttle(() => {
    if (window.innerHeight + window.scrollY >= document.body.offsetHeight) {
      setOffset(offset + limit); // Load more events
    }
  }, 200); // Fire every 200ms
  ```

### Client-Side Caching with Service Workers

- Challenge: Repeatedly fetching event data, especially for large playlists, can cause latency in the user experience.
- Solution: Use Service Workers to cache frequently accessed resources (e.g., events) on the client-side.
  - Benefit: Reduces the number of network requests by serving cached data when possible.
- Example: Implement a service worker that caches event data and serves it from the cache when available.

### Progressive Loading and Skeleton Screens

- Challenge: Users may perceive the UI as slow if large datasets are loaded without any visual feedback.
- Solution: Implement skeleton screens to show placeholder content while data is being fetched.
  - Benefit: Improves perceived performance by giving users feedback that the data is being loaded.
- Example:

  ```jsx
  function SkeletonEvent() {
    return <div className="skeleton-event">Loading...</div>;
  }

  function Playlist() {
    const [loading, setLoading] = useState(true);

    useEffect(() => {
      // Simulate data fetching
      setTimeout(() => {
        setLoading(false);
      }, 2000); // Example delay
    }, []);

    return <div>{loading ? <SkeletonEvent /> : <EventList />}</div>;
  }
  ```

### Optimizing Image and Media Delivery

- Challenge: If your events contain images or media, loading them without optimization can severely impact performance.
- Solution: Use lazy loading for images and media files, and serve them via a Content Delivery Network (CDN).
  - Benefit: Reduces initial page load times and speeds up media delivery by caching assets close to users.
  ```jsx
  <img src="image.jpg" loading="lazy" alt="Event" />
  ```

### Batch WebSocket Messages

- Challenge: Sending individual WebSocket messages for each update can flood the network and slow down client updates.
- Solution: Batch WebSocket messages by aggregating multiple updates into a single message before sending it to the client.

  - Benefit: Reduces the number of WebSocket messages sent, improving performance.

  ```ts
  let messageBuffer = [];
  socket.on("event-updated", (data) => {
    messageBuffer.push(data);
  });

  setInterval(() => {
    if (messageBuffer.length > 0) {
      socket.emit("update-playlist", messageBuffer);
      messageBuffer = [];
    }
  }, 1000); // Batch updates every second
  ```

## Summary of Optimizations

- Backend Performance:
  - Implement caching using Redis.
  - Partition and shard database tables to improve query efficiency.
  - Offload long-running tasks to background jobs.
  - Use eventual consistency for non-critical operations.
  - Perform bulk inserts/updates when possible.
- Frontend Real-Time User Experience:
  - Use delta updates in WebSocket communication.
  - Throttle and debounce scroll events for infinite scrolling.
  - Utilize service workers for client-side caching.
  - Implement skeleton screens to improve perceived performance.
  - Optimize image and media loading with lazy loading and CDNs.
  - Batch WebSocket messages to reduce network load.

These optimizations can significantly improve both the backend system performance and the real-time user experience on the frontend. By implementing them, you ensure that the system scales efficiently as the number of users and events increases, while also providing a smooth and responsive experience for users interacting with the playlist.
