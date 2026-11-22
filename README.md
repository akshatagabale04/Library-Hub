# Library-Hub

A library website where members can search the catalogue, borrow books, return them and see late fines. It is built as a microservices system: a React frontend talks to an API gateway, which routes requests to small Spring Boot services. Each service owns its own PostgreSQL database, and RabbitMQ carries asynchronous "email" notifications.

Tech stack: React (Vite) · Java 17 · Spring Boot 3.3 · Spring Cloud Gateway · PostgreSQL 16 · RabbitMQ · Docker Compose · Maven multi-module

<!-- Add screenshots after you run the app: ![Catalogue](docs/catalogue.png) ![My loans](docs/my-loans.png) -->
Features
Browse 12 seeded books, search by title or author, and filter by category
Join the library or sign in with an email address (demo, no password yet)
Borrow and return books, with live copy counts
Library rules: 14-day loans, maximum 3 active loans per member, one copy of a title per member, and a fine of ₹5 per late day
No double lending: the last copy of a book can never be lent to two people at once
Background notifications: loan and return events are published to RabbitMQ and "emailed" (written to the log) by a separate service

Architecture
React UI (:3000) -> API Gateway (:8080) -> catalog-service   (:8081) -> catalog_db
                                        -> loan-service      (:8082) -> loan_db
                                        -> member-service    (:8083) -> member_db
                                        -> inventory-service (:8084) -> inventory_db

loan-service --publishes loan.created / loan.returned--> RabbitMQ --> notification-service (:8085)
Service	Port	Database	Job
frontend (React)	3000	none	Catalogue, search, borrow, my loans, join and sign in
gateway	8080	none	Single entry point for /api/books, /api/loans, /api/members, /api/inventory, plus CORS
catalog-service	8081	catalog_db	Books, search by title or author, filter by category
loan-service	8082	loan_db	Borrow and return, limits, due dates, fines; publishes events
member-service	8083	member_db	Join and look up members by email
inventory-service	8084	inventory_db	Copies per book, with an atomic checkout
notification-service	8085	none	Consumes loan.created and loan.returned, logs the email

Golden rule: no service reads another service's database. They talk through REST calls or RabbitMQ events.

How a borrow works
The UI sends POST /api/loans with the member's email and the book id.
The loan service checks the member exists, checks the 3-book limit and the duplicate-title rule, and asks the catalog service for the book.
The inventory service takes one copy using UPDATE copies SET available = available - 1 WHERE book_id = ? AND available > 0. If no row changes, nobody gets the book. This is what prevents two people from borrowing the last copy.
The loan is saved with a due date 14 days away, and a loan.created event goes to RabbitMQ.
The notification service picks up the event and logs the email.
Getting started

Option 1: Docker (everything in one command)
Requires Docker Desktop.

bash
docker compose up --build -d
docker compose ps -a        # wait until all services are Up (about a minute)
App: http://localhost:3000
RabbitMQ dashboard: http://localhost:15672 (login guest / guest)
Watch the emails: docker compose logs -f notification-service
Stop: docker compose down (add -v to wipe the data)

Option 2: Develop from your PC
Requires Java 17, Maven 3.9+ and Node 20+.

bash
docker compose up postgres rabbitmq -d     # PostgreSQL is published on port 5433

mvn -pl catalog-service spring-boot:run    # then in separate terminals:
mvn -pl inventory-service spring-boot:run
mvn -pl member-service spring-boot:run
mvn -pl notification-service spring-boot:run
mvn -pl loan-service spring-boot:run       # start after catalog, member and inventory
mvn -pl gateway spring-boot:run

cd frontend && npm install && npm run dev  # http://localhost:3000

Services connect to localhost:5433 by default (user and password library). Each service sets its own time zone to UTC, so the Asia/Calcutta PostgreSQL error does not occur.

Try it
Join with any name and email, open Catalogue and borrow a book.
Pride and Prejudice has only 1 copy. Borrow it, then join with a second email and try again. You get "No copies available".
Borrow 3 books, then try a 4th. You get the limit message.
Open My loans and choose Return 5 days late. The fine shows as 5 days × ₹5 = ₹25.

API reference
Everything goes through the gateway on port 8080.

Method	Endpoint	Purpose
GET	/api/books?category=&q=	List or search books
GET	/api/books/{id}	One book
POST	/api/members	Join (name, email). Returns 409 if the email already exists
GET	/api/members?email=	Sign in by email
POST	/api/loans	Borrow (email, bookId)
GET	/api/loans?email=	A member's loans, newest first
POST	/api/loans/{id}/return?demoDaysLate=0	Return a book. demoDaysLate lets you test fines without waiting
GET	/api/inventory	Copies total and available for every book
Project structure
gateway/               Spring Cloud Gateway routes
catalog-service/       Books and search
loan-service/          Borrow, return, rules, fines, event publishing
member-service/        Members
inventory-service/     Copy counts with atomic checkout
notification-service/  RabbitMQ consumer
frontend/              React app
db/init.sql            Creates the four databases
docker-compose.yml
Not included yet

Passwords and JWT at the gateway, reservations and waiting lists, librarian and admin screens, a real email provider, and automated tests.
