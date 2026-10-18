# Course Reflection

The most challenging concept in the course was designing distributed system architectures and managing high-concurrency data persistence across relational databases and caching layers. Concepts like optimistic concurrency locking (`version` fields), mitigating race conditions during traffic spikes, and maintaining cache consistency required shifting from basic web development into deep systems engineering. I overcame this by mapping request lifecycles step-by-step from client to database and testing raw SQL transactions locally in PostgreSQL.

Based on project review feedback, I would improve my capstone project by enhancing client-side resilience during network delays. Specifically, I would implement optimistic UI updates for instant visual feedback when adding notes, add dynamic CSS skeleton loaders instead of basic text status messages, and incorporate exponential backoff retry logic for transient API network failures in `api.js`.

Next, I will focus on mastering **TypeScript**, **react-native**, and **Docker**. Mastering TypeScript will enable me to write type-safe code across full-stack repositories, while learning Docker will allow me to containerize web applications and manage multi-container environments using Docker Compose efficiently.
