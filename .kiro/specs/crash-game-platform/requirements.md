# Requirements Document

## Introduction

The Crash Game Platform is a full-stack multiplayer casino gaming system where players place bets on a continuously rising multiplier that "crashes" at an unpredictable point. Players must cash out before the crash to win their bet multiplied by the current multiplier. The platform implements provably fair gaming mechanics, real-time synchronization across multiple clients, and a microservices architecture with separate Game and Wallet services.

## Glossary

- **Game_Service**: The NestJS microservice responsible for managing round lifecycle, betting logic, crash point generation, and WebSocket communication
- **Wallet_Service**: The NestJS microservice responsible for managing player wallets, balance operations, and transaction history
- **API_Gateway**: Kong gateway that routes all client requests to appropriate backend services
- **Auth_Service**: Keycloak authentication service providing OIDC/JWT-based authentication
- **Message_Broker**: RabbitMQ or AWS SQS service for asynchronous inter-service communication
- **Player**: An authenticated user participating in the crash game
- **Round**: A complete game cycle consisting of betting phase, active phase, and crash
- **Betting_Phase**: The configurable time window (default 10 seconds) when players can place bets
- **Active_Phase**: The period when the multiplier is rising and players can cash out
- **Multiplier**: The continuously rising value starting at 1.00x that determines payout amounts
- **Crash_Point**: The predetermined multiplier value at which the round ends
- **Cash_Out**: The action of a player claiming their winnings during the active phase
- **Bet**: A monetary wager placed by a player during the betting phase
- **Payout**: The amount credited to a player's wallet (bet amount × cash out multiplier)
- **Provably_Fair_Algorithm**: The cryptographic system using hash chains and HMAC to ensure game integrity
- **Seed**: A cryptographic value used in the provably fair algorithm to generate crash points
- **Frontend_Client**: The React/Next.js web application providing the user interface
- **WebSocket_Connection**: The real-time bidirectional communication channel between client and Game_Service
- **Balance**: The amount of currency available in a player's wallet (stored as integer cents)
- **Transaction**: A credit or debit operation on a player's wallet
- **Round_History**: The record of past rounds including crash points and player outcomes
- **Auto_Cashout**: An optional feature allowing players to set a target multiplier for automatic cash out
- **Auto_Bet**: An optional feature allowing players to automatically place bets using predefined strategies

## Requirements

### Requirement 1: Player Authentication and Authorization

**User Story:** As a player, I want to authenticate securely using industry-standard protocols, so that my account and funds are protected.

#### Acceptance Criteria

1. THE API_Gateway SHALL validate JWT tokens issued by Auth_Service for all protected endpoints
2. WHEN a player attempts to access protected resources without a valid JWT, THEN THE API_Gateway SHALL return HTTP 401 Unauthorized
3. THE Auth_Service SHALL issue JWT tokens containing player identity and roles after successful authentication
4. THE Game_Service SHALL extract player identity from validated JWT tokens for all game operations
5. THE Wallet_Service SHALL extract player identity from validated JWT tokens for all wallet operations

### Requirement 2: Round Lifecycle Management

**User Story:** As the system, I want to manage round lifecycles automatically, so that the game progresses continuously without manual intervention.

#### Acceptance Criteria

1. THE Game_Service SHALL create a new round with a betting phase immediately after the previous round ends
2. WHEN the betting phase duration expires, THE Game_Service SHALL transition the round to active phase
3. WHILE the round is in active phase, THE Game_Service SHALL increment the multiplier continuously from 1.00x
4. WHEN the multiplier reaches the crash point, THE Game_Service SHALL end the round and transition to betting phase for the next round
5. THE Game_Service SHALL generate the crash point using the Provably_Fair_Algorithm before the round starts
6. THE Game_Service SHALL store round state including round ID, crash point, start time, and end time in the database

### Requirement 3: Betting Phase Operations

**User Story:** As a player, I want to place bets during the betting phase, so that I can participate in the upcoming round.

#### Acceptance Criteria

1. WHILE the round is in betting phase, THE Game_Service SHALL accept bet placement requests from authenticated players
2. WHEN a player places a bet, THE Game_Service SHALL validate the bet amount is between 1.00 and 1000.00 currency units
3. WHEN a player places a bet, THE Game_Service SHALL verify the player has not already placed a bet in the current round
4. WHEN a player places a bet, THE Game_Service SHALL send a debit request to Wallet_Service via Message_Broker
5. IF the Wallet_Service confirms insufficient balance, THEN THE Game_Service SHALL reject the bet and notify the player
6. WHEN the Wallet_Service confirms successful debit, THE Game_Service SHALL record the bet in the database
7. WHEN a bet is successfully placed, THE Game_Service SHALL broadcast the bet information to all connected clients via WebSocket
8. IF a player attempts to place a bet during active phase, THEN THE Game_Service SHALL reject the bet with an appropriate error message

### Requirement 4: Cash Out Operations

**User Story:** As a player, I want to cash out during the active phase, so that I can secure my winnings before the crash.

#### Acceptance Criteria

1. WHILE the round is in active phase, THE Game_Service SHALL accept cash out requests from players with active bets
2. WHEN a player requests cash out, THE Game_Service SHALL calculate payout as bet amount multiplied by current multiplier
3. WHEN a player requests cash out, THE Game_Service SHALL record the cash out time and multiplier in the database
4. WHEN a player cashes out, THE Game_Service SHALL send a credit request to Wallet_Service via Message_Broker with the calculated payout
5. WHEN the Wallet_Service confirms successful credit, THE Game_Service SHALL mark the bet as cashed out
6. WHEN a cash out is successful, THE Game_Service SHALL broadcast the cash out information to all connected clients via WebSocket
7. IF a player attempts to cash out without an active bet in the current round, THEN THE Game_Service SHALL reject the request
8. IF a player attempts to cash out after already cashing out in the current round, THEN THE Game_Service SHALL reject the request
9. IF a player attempts to cash out during betting phase, THEN THE Game_Service SHALL reject the request

### Requirement 5: Wallet Balance Management

**User Story:** As a player, I want my wallet balance to be accurately maintained, so that I can trust the platform with my funds.

#### Acceptance Criteria

1. THE Wallet_Service SHALL store all balance amounts as integers representing the smallest currency unit (cents)
2. WHEN the Wallet_Service receives a debit request, THE Wallet_Service SHALL verify sufficient balance exists
3. WHEN the Wallet_Service processes a debit, THE Wallet_Service SHALL atomically decrease the balance and create a transaction record
4. WHEN the Wallet_Service processes a credit, THE Wallet_Service SHALL atomically increase the balance and create a transaction record
5. THE Wallet_Service SHALL ensure balance never becomes negative through database constraints
6. WHEN a balance operation completes, THE Wallet_Service SHALL publish a confirmation event to Message_Broker
7. IF a debit request exceeds available balance, THEN THE Wallet_Service SHALL reject the operation and publish a failure event
8. THE Wallet_Service SHALL provide an endpoint to query current balance for authenticated players

### Requirement 6: Real-Time Client Synchronization

**User Story:** As a player, I want to see real-time updates of the game state, so that I can make informed decisions about when to cash out.

#### Acceptance Criteria

1. THE Game_Service SHALL establish WebSocket connections with Frontend_Client instances for authenticated players
2. WHEN the betting phase starts, THE Game_Service SHALL broadcast a round start event to all connected clients
3. WHILE the round is in active phase, THE Game_Service SHALL broadcast multiplier updates at least 10 times per second to all connected clients
4. WHEN any player places a bet, THE Game_Service SHALL broadcast the bet event to all connected clients
5. WHEN any player cashes out, THE Game_Service SHALL broadcast the cash out event to all connected clients
6. WHEN the round crashes, THE Game_Service SHALL broadcast the crash event with final crash point to all connected clients
7. WHEN a WebSocket connection is established, THE Game_Service SHALL send the current round state to the newly connected client
8. IF a WebSocket connection is lost, THE Frontend_Client SHALL attempt to reconnect automatically

### Requirement 7: Provably Fair Crash Point Generation

**User Story:** As a player, I want the crash points to be provably fair and verifiable, so that I can trust the game is not rigged.

#### Acceptance Criteria

1. THE Game_Service SHALL generate crash points using a cryptographic hash chain with server seed and client seed
2. THE Game_Service SHALL publish the hash of the server seed before the round starts
3. WHEN a round ends, THE Game_Service SHALL reveal the server seed used for that round
4. THE Game_Service SHALL provide an endpoint to verify crash point calculation given server seed, client seed, and nonce
5. THE Game_Service SHALL use HMAC-SHA256 for hash chain generation
6. THE Game_Service SHALL store server seed, client seed, nonce, and crash point hash for each round
7. FOR ALL rounds, verifying the crash point using the revealed seeds SHALL produce the same crash point value (round-trip property)

### Requirement 8: Provably Fair Verification Endpoint

**User Story:** As a player, I want to verify past round results, so that I can confirm the game was fair.

#### Acceptance Criteria

1. THE Game_Service SHALL provide an HTTP endpoint accepting round ID, server seed, client seed, and nonce as inputs
2. WHEN the verification endpoint receives valid inputs, THE Game_Service SHALL recalculate the crash point using the Provably_Fair_Algorithm
3. WHEN the verification endpoint completes calculation, THE Game_Service SHALL return the calculated crash point and comparison with stored crash point
4. THE Game_Service SHALL provide documentation explaining the provably fair algorithm and verification process
5. IF the verification endpoint receives invalid inputs, THEN THE Game_Service SHALL return HTTP 400 with descriptive error message

### Requirement 9: Monetary Precision and Accuracy

**User Story:** As the system, I want to handle all monetary values with perfect precision, so that no rounding errors occur in financial calculations.

#### Acceptance Criteria

1. THE Wallet_Service SHALL store all monetary amounts as BIGINT or NUMERIC types representing smallest currency unit
2. THE Game_Service SHALL store all bet amounts and payouts as BIGINT or NUMERIC types representing smallest currency unit
3. THE Game_Service SHALL perform all payout calculations using integer arithmetic or fixed-point decimal arithmetic
4. THE API_Gateway SHALL validate all monetary inputs from clients are properly formatted before forwarding to services
5. THE Frontend_Client SHALL display monetary amounts converted from smallest currency unit to standard currency format
6. FOR ALL payout calculations, converting to smallest unit, calculating, and converting back SHALL preserve exact values (no precision loss)

### Requirement 10: Database Isolation and Service Boundaries

**User Story:** As a system architect, I want each microservice to have its own database, so that services remain loosely coupled and independently deployable.

#### Acceptance Criteria

1. THE Game_Service SHALL connect exclusively to its own PostgreSQL database instance
2. THE Wallet_Service SHALL connect exclusively to its own PostgreSQL database instance
3. THE Game_Service SHALL NOT directly query Wallet_Service database tables
4. THE Wallet_Service SHALL NOT directly query Game_Service database tables
5. WHEN services need data from each other, THE services SHALL communicate via Message_Broker events or HTTP APIs
6. THE Game_Service database SHALL store rounds, bets, crash points, and provably fair seeds
7. THE Wallet_Service database SHALL store wallets, balances, and transactions

### Requirement 11: Asynchronous Inter-Service Communication

**User Story:** As the system, I want services to communicate asynchronously, so that the system remains responsive and resilient to temporary service failures.

#### Acceptance Criteria

1. THE Game_Service SHALL publish bet debit requests to Message_Broker when players place bets
2. THE Wallet_Service SHALL consume bet debit requests from Message_Broker and process them
3. THE Wallet_Service SHALL publish debit confirmation or failure events to Message_Broker
4. THE Game_Service SHALL consume wallet operation results from Message_Broker
5. THE Game_Service SHALL publish payout credit requests to Message_Broker when players cash out
6. THE Wallet_Service SHALL consume payout credit requests from Message_Broker and process them
7. WHEN Message_Broker is temporarily unavailable, THE services SHALL retry message publishing with exponential backoff
8. THE services SHALL implement idempotent message handlers to safely process duplicate messages

### Requirement 12: API Gateway Routing and Rate Limiting

**User Story:** As the system, I want all client requests to flow through a centralized gateway, so that I can apply consistent security, routing, and rate limiting policies.

#### Acceptance Criteria

1. THE API_Gateway SHALL route requests to /api/game/* to Game_Service
2. THE API_Gateway SHALL route requests to /api/wallet/* to Wallet_Service
3. THE API_Gateway SHALL route WebSocket upgrade requests to Game_Service
4. WHERE rate limiting is enabled, THE API_Gateway SHALL limit requests to 100 per minute per authenticated player
5. WHEN rate limit is exceeded, THE API_Gateway SHALL return HTTP 429 Too Many Requests
6. THE API_Gateway SHALL add correlation IDs to all forwarded requests for distributed tracing
7. THE API_Gateway SHALL log all requests with timestamp, player ID, endpoint, and response status

### Requirement 13: Frontend Real-Time Visualization

**User Story:** As a player, I want to see an animated graph of the rising multiplier, so that I can visually track the game progress and make cash out decisions.

#### Acceptance Criteria

1. THE Frontend_Client SHALL render a continuously updating line graph showing multiplier over time
2. WHILE receiving multiplier updates via WebSocket, THE Frontend_Client SHALL animate the graph smoothly
3. THE Frontend_Client SHALL display the current multiplier value prominently with at least 2 decimal places
4. WHEN the round crashes, THE Frontend_Client SHALL visually indicate the crash with animation and sound effect
5. THE Frontend_Client SHALL display a countdown timer during the betting phase
6. THE Frontend_Client SHALL display a list of active players and their bet amounts during the round
7. WHEN a player cashes out, THE Frontend_Client SHALL display the cash out multiplier next to the player's name
8. THE Frontend_Client SHALL display round history showing at least the last 10 crash points

### Requirement 14: Frontend Betting Controls

**User Story:** As a player, I want intuitive betting controls, so that I can quickly place bets and cash out during the game.

#### Acceptance Criteria

1. THE Frontend_Client SHALL provide an input field for entering bet amounts with validation
2. THE Frontend_Client SHALL provide preset bet amount buttons (e.g., 10, 50, 100, 500)
3. WHILE the round is in betting phase, THE Frontend_Client SHALL enable the place bet button
4. WHILE the round is in active phase AND the player has an active bet, THE Frontend_Client SHALL enable the cash out button
5. WHEN the player clicks place bet, THE Frontend_Client SHALL send a bet request to Game_Service via API_Gateway
6. WHEN the player clicks cash out, THE Frontend_Client SHALL send a cash out request to Game_Service via WebSocket
7. THE Frontend_Client SHALL display the player's current wallet balance
8. THE Frontend_Client SHALL disable betting controls when the player has insufficient balance

### Requirement 15: Domain-Driven Design Architecture

**User Story:** As a developer, I want the codebase to follow DDD principles, so that the business logic is clearly separated and maintainable.

#### Acceptance Criteria

1. THE Game_Service SHALL organize code into domain layer, application layer, and infrastructure layer
2. THE Wallet_Service SHALL organize code into domain layer, application layer, and infrastructure layer
3. THE Game_Service SHALL define aggregate roots for Round and Bet entities
4. THE Wallet_Service SHALL define aggregate roots for Wallet and Transaction entities
5. THE services SHALL implement repository patterns for data persistence abstraction
6. THE services SHALL implement domain events for communicating state changes
7. THE services SHALL keep domain logic free from infrastructure dependencies

### Requirement 16: Docker Compose Development Environment

**User Story:** As a developer, I want to start the entire platform with a single command, so that I can quickly set up a development environment.

#### Acceptance Criteria

1. THE project SHALL provide a docker-compose.yml file defining all services
2. WHEN a developer runs "bun run docker:up", THE system SHALL start PostgreSQL, RabbitMQ, Kong, Keycloak, Game_Service, Wallet_Service, and Frontend_Client
3. THE docker-compose.yml SHALL configure service dependencies to ensure proper startup order
4. THE docker-compose.yml SHALL expose appropriate ports for local development access
5. THE docker-compose.yml SHALL configure environment variables for service configuration
6. THE docker-compose.yml SHALL mount source code volumes for hot-reloading during development
7. WHEN all services start successfully, THE Frontend_Client SHALL be accessible at http://localhost:3000

### Requirement 17: Comprehensive Testing Strategy

**User Story:** As a developer, I want comprehensive test coverage, so that I can confidently refactor and extend the codebase.

#### Acceptance Criteria

1. THE Game_Service SHALL include unit tests for domain logic with at least 80% code coverage
2. THE Wallet_Service SHALL include unit tests for domain logic with at least 80% code coverage
3. THE Game_Service SHALL include integration tests for API endpoints
4. THE Wallet_Service SHALL include integration tests for API endpoints
5. THE project SHALL include end-to-end tests simulating complete player workflows
6. THE tests SHALL use test doubles (mocks/stubs) for external dependencies
7. THE project SHALL provide a command to run all tests with a single invocation

### Requirement 18: Auto Cashout Feature

**User Story:** As a player, I want to set a target multiplier for automatic cash out, so that I can secure winnings at my desired multiplier without manual intervention.

#### Acceptance Criteria

1. WHERE auto cashout is enabled, THE Frontend_Client SHALL provide an input field for target multiplier
2. WHERE auto cashout is enabled AND a player sets a target multiplier, THE Game_Service SHALL store the auto cashout preference with the bet
3. WHERE auto cashout is enabled, WHILE the round is in active phase, THE Game_Service SHALL automatically cash out the player when the multiplier reaches the target
4. WHERE auto cashout is enabled, WHEN auto cashout triggers, THE Game_Service SHALL process the cash out identically to manual cash out
5. WHERE auto cashout is enabled, THE Frontend_Client SHALL visually indicate that auto cashout is set for the player's bet

### Requirement 19: Auto Bet Feature

**User Story:** As a player, I want to automatically place bets using predefined strategies, so that I can participate in multiple rounds without manual bet placement.

#### Acceptance Criteria

1. WHERE auto bet is enabled, THE Frontend_Client SHALL provide controls to enable/disable auto bet
2. WHERE auto bet is enabled, THE Frontend_Client SHALL provide strategy selection (fixed amount, Martingale)
3. WHERE auto bet is enabled AND Martingale strategy is selected, THE Game_Service SHALL double the bet amount after each loss
4. WHERE auto bet is enabled AND fixed amount strategy is selected, THE Game_Service SHALL place the same bet amount each round
5. WHERE auto bet is enabled, WHEN a new betting phase starts, THE Game_Service SHALL automatically place a bet according to the selected strategy
6. WHERE auto bet is enabled, IF the player has insufficient balance, THEN THE Game_Service SHALL disable auto bet and notify the player

### Requirement 20: Observability and Monitoring

**User Story:** As a system operator, I want comprehensive observability, so that I can monitor system health and diagnose issues quickly.

#### Acceptance Criteria

1. WHERE observability is enabled, THE Game_Service SHALL export metrics to Prometheus
2. WHERE observability is enabled, THE Wallet_Service SHALL export metrics to Prometheus
3. WHERE observability is enabled, THE services SHALL instrument code with OpenTelemetry for distributed tracing
4. WHERE observability is enabled, THE services SHALL export traces to a compatible backend
5. WHERE observability is enabled, THE project SHALL provide Grafana dashboards for visualizing metrics
6. WHERE observability is enabled, THE services SHALL log structured JSON logs with correlation IDs
7. WHERE observability is enabled, THE services SHALL expose health check endpoints for liveness and readiness probes

### Requirement 21: Leaderboard Feature

**User Story:** As a player, I want to see a leaderboard of top players, so that I can compare my performance with others.

#### Acceptance Criteria

1. WHERE leaderboard is enabled, THE Game_Service SHALL track total winnings for each player
2. WHERE leaderboard is enabled, THE Game_Service SHALL provide an endpoint to query top players by total winnings
3. WHERE leaderboard is enabled, THE Frontend_Client SHALL display a leaderboard showing player names and total winnings
4. WHERE leaderboard is enabled, THE leaderboard SHALL update after each round completes
5. WHERE leaderboard is enabled, THE leaderboard SHALL display at least the top 10 players

### Requirement 22: Configuration Management

**User Story:** As a system operator, I want to configure game parameters without code changes, so that I can tune the game experience.

#### Acceptance Criteria

1. THE Game_Service SHALL load betting phase duration from environment variables or configuration files
2. THE Game_Service SHALL load minimum bet amount from environment variables or configuration files
3. THE Game_Service SHALL load maximum bet amount from environment variables or configuration files
4. THE Game_Service SHALL load multiplier update frequency from environment variables or configuration files
5. THE Game_Service SHALL validate all configuration values at startup
6. IF configuration values are invalid, THEN THE Game_Service SHALL fail to start with descriptive error messages

### Requirement 23: Error Handling and Recovery

**User Story:** As a player, I want the system to handle errors gracefully, so that temporary issues don't result in lost funds or unfair outcomes.

#### Acceptance Criteria

1. WHEN the Wallet_Service is temporarily unavailable, THE Game_Service SHALL queue debit and credit requests for retry
2. WHEN a database transaction fails, THE services SHALL rollback all changes and return an error to the client
3. WHEN a WebSocket connection fails during a round, THE Frontend_Client SHALL reconnect and request current round state
4. IF a player's cash out request is received after the crash, THEN THE Game_Service SHALL reject the request and preserve the loss
5. WHEN an unexpected error occurs, THE services SHALL log the error with full context and return a generic error message to clients
6. THE services SHALL implement circuit breakers for external service calls to prevent cascade failures

### Requirement 24: Security and Input Validation

**User Story:** As a system operator, I want all inputs validated and sanitized, so that the system is protected from malicious actors.

#### Acceptance Criteria

1. THE API_Gateway SHALL validate all request payloads against defined schemas before forwarding
2. THE Game_Service SHALL validate all bet amounts are positive numbers within allowed range
3. THE Game_Service SHALL validate all player IDs match the authenticated JWT subject
4. THE Wallet_Service SHALL validate all transaction amounts are positive integers
5. THE services SHALL sanitize all string inputs to prevent SQL injection attacks
6. THE services SHALL use parameterized queries for all database operations
7. THE services SHALL implement CORS policies to restrict frontend origins

### Requirement 25: Performance and Scalability

**User Story:** As a system operator, I want the platform to handle high concurrent player loads, so that the game remains responsive during peak times.

#### Acceptance Criteria

1. THE Game_Service SHALL support at least 1000 concurrent WebSocket connections
2. THE Game_Service SHALL broadcast multiplier updates to all connected clients within 100 milliseconds
3. THE Wallet_Service SHALL process balance operations within 200 milliseconds at the 95th percentile
4. THE services SHALL use database connection pooling to efficiently manage database connections
5. THE services SHALL implement caching for frequently accessed read-only data
6. THE Game_Service SHALL use efficient data structures for tracking active bets in memory
7. THE system SHALL support horizontal scaling by running multiple instances of each service behind load balancers
