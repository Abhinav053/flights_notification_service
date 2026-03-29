# 📧 Reminder Service Application

## Overview

**Reminder Service** is a robust and scalable Node.js application designed to manage email notifications and reminders. It provides a complete solution for creating, storing, and automatically sending scheduled email reminders with status tracking. The system use event-driven architecture with message queues to handle asynchronous processing, making it ideal for microservices environments.

---

## 🎯 Purpose & Use Cases

This service is built to handle:
- **Email Reminder Creation**: Create new email reminders with specific send times
- **Scheduled Email Delivery**: Automatically send emails at the specified notification time
- **Status Tracking**: Track the status of each reminder (PENDING, SUCCESS, FAILED)
- **Event-Driven Processing**: Receive messages from other services via RabbitMQ and process them asynchronously
- **Microservices Integration**: Works as part of a larger microservices architecture (like a flight booking system)

**Real-world Scenarios:**
- Flight booking reminders ("Your flight departs in 24 hours")
- Appointment reminders
- Order status notifications
- Event countdown notifications

---

## 📋 Project Structure

```
ReminderService/
├── .env                          # Environment variables configuration
├── .gitignore                    # Git ignore rules
├── package.json                  # Dependencies and scripts
├── src/
│   ├── index.js                  # Application entry point - starts server
│   ├── config/
│   │   ├── config.json          # Database configuration for different environments
│   │   ├── emailConfig.js       # Nodemailer SMTP configuration
│   │   └── serverConfig.js      # Server & message broker configuration
│   ├── models/
│   │   ├── index.js             # Sequelize database connection & model initialization
│   │   └── notificationticket.js # NotificationTicket database model
│   ├── controllers/
│   │   └── ticket-controller.js # Request handlers for ticket operations
│   ├── services/
│   │   └── email-service.js     # Business logic for email operations
│   ├── repository/
│   │   └── ticket-repository.js # Database query operations (Data Access Layer)
│   ├── migrations/
│   │   └── 20230101091853-create-notification-ticket.js # Database schema creation
│   └── utils/
│       ├── messageQueue.js      # RabbitMQ message broker functions
│       └── job.js               # Cron job scheduler for periodic email sending
└── node_modules/                # Installed dependencies (generated)
```

---

## 🏗️ Architecture & Data Flow

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Reminder Service Architecture                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
            ┌───────▼────────┐   ┌──────▼────────┐
            │   API Request  │   │   RabbitMQ    │
            │  (HTTP POST)   │   │   Messages    │
            └───────┬────────┘   └──────┬────────┘
                    │                   │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Controller       │
                    │ (ticket-controller)│
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Service Layer    │
                    │ (email-service)    │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Repository       │
                    │ (ticket-repository)│
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────────┐
                    │   MySQL Database       │
                    │ (NotificationTicket)   │
                    └────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Scheduler (Cron) │
                    │   Every 2 minutes  │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Nodemailer       │
                    │   Gmail SMTP       │
                    └────────────────────┘
```

### Data Flow Steps

1. **Ticket Creation**: Data arrives via HTTP POST or RabbitMQ message
2. **Storage**: Ticket saved to MySQL database with PENDING status
3. **Scheduling**: Cron job runs every 2 minutes to check for pending emails
4. **Email Sending**: Emails with notification time ≤ current time are sent
5. **Status Update**: Database record updated to SUCCESS or FAILED

---

## 🔧 Key Components Explained

### 1. **Entry Point** (`src/index.js`)

```javascript
// Creates Express server
// Sets up body parser middleware for JSON requests
// Registers API routes
// Creates RabbitMQ channel for message subscription
// Starts listening on configured PORT
```

**What it does:**
- Initializes the Express application
- Configures middleware to parse JSON request bodies
- Sets up the POST endpoint: `/api/v1/tickets`
- Connects to RabbitMQ and subscribes to messages with binding key `booking.reminder`
- Starts the server on port 3003

---

### 2. **Configuration Files** (`src/config/`)

#### **serverConfig.js**
Loads all environment variables from `.env` file:
- `PORT`: Server port (3003)
- `EMAIL_ID`: Sender's Gmail address
- `EMAIL_PASS`: Gmail app password
- `MESSAGE_BROKER_URL`: RabbitMQ connection URL
- `EXCHANGE_NAME`: Message exchange name (booking_exchange)
- `REMINDER_BINDING_KEY`: Message routing key (booking.reminder)

#### **emailConfig.js**
Sets up Nodemailer transporter for sending emails via Gmail SMTP:
- Uses Gmail service with authentication
- Configures sender email and password
- Returns configured transporter object

#### **config.json**
Database configuration for different environments:
```json
{
  "development": {
    "database": "flights_REMINDER_DB_",
    "username": "root",
    "password": "root",
    "host": "127.0.0.1",
    "dialect": "mysql"
  }
  // test and production configs also present
}
```

---

### 3. **Data Model** (`src/models/notificationticket.js`)

Defines the NotificationTicket Sequelize model with these fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Primary key (auto-increment) |
| `subject` | STRING | Email subject line |
| `content` | STRING | Email body text |
| `recepientEmail` | STRING | Recipient email address |
| `status` | ENUM | One of: PENDING, SUCCESS, FAILED |
| `notificationTime` | DATE | When the email should be sent |
| `createdAt` | DATE | Record creation timestamp |
| `updatedAt` | DATE | Last update timestamp |

---

### 4. **Repository Layer** (`src/repository/ticket-repository.js`)

Implements the Data Access Object (DAO) pattern for database operations:

**Methods:**

```javascript
// Get all notification tickets
getAll()

// Create a new notification ticket
create(data)

// Get pending tickets that should be sent now
get(filter)  
// Filters by: status = "PENDING" AND notificationTime <= NOW

// Update ticket status (after email sent)
update(ticketId, data)
```

**Why Repository Layer?**
- Centralizes all database queries
- Makes code testable and maintainable
- Separates business logic from data access

---

### 5. **Service Layer** (`src/services/email-service.js`)

Contains core business logic for email operations:

**Key Functions:**

```javascript
// Low-level email sending via Nodemailer
sendBasicEmail(mailFrom, mailTo, mailSubject, mailBody)

// Get all pending emails ready to send
fetchPendingEmails(timestamp)

// Update a ticket's status in database
updateTicket(ticketId, data)

// Create a new notification record
createNotification(data)

// Event handler for RabbitMQ messages
subscribeEvents(payload)
// Handles: CREATE_TICKET, SEND_BASIC_MAIL events
```

---

### 6. **Controller Layer** (`src/controllers/ticket-controller.js`)

Handles HTTP requests and returns responses:

**Endpoint:**
- `POST /api/v1/tickets` - Creates a new reminder ticket

**Request Format:**
```json
{
  "subject": "Flight Reminder",
  "content": "Your flight departs in 24 hours",
  "recepientEmail": "user@example.com",
  "notificationTime": "2023-12-25T10:00:00Z"
}
```

**Success Response (201):**
```json
{
  "success": true,
  "data": { /* created ticket object */ },
  "err": {},
  "message": "Successfully registered an email reminder"
}
```

**Error Response (500):**
```json
{
  "success": false,
  "data": null,
  "err": { /* error details */ },
  "message": "Unable to register an email reminder"
}
```

---

### 7. **Message Queue** (`src/utils/messageQueue.js`)

Integrates with RabbitMQ for asynchronous message processing:

**Key Functions:**

```javascript
// Connect to RabbitMQ and create channel
createChannel()
// Returns a channel configured with direct exchange

// Subscribe to messages on a specific binding key
subscribeMessage(channel, service, binding_key)
// Listens on REMINDER_QUEUE
// Parses JSON payload and calls service function
// Auto-acknowledges message after processing

// Publish messages to exchange
publishMessage(channel, binding_key, message)
// Sends Buffer message to specified routing key
```

**Message Flow:**
1. External service publishes message to `booking_exchange` with key `booking.reminder`
2. Message arrives in `REMINDER_QUEUE`
3. Service receives and parses JSON payload
4. Calls appropriate email service handler
5. Auto-acknowledges receipt to RabbitMQ

---

### 8. **Scheduled Jobs** (`src/utils/job.js`)

Uses cron scheduling to periodically send pending emails:

**Cron Schedule:** `*/2 * * * *` = Every 2 minutes

**Job Workflow:**
1. Fetches all tickets with status = "PENDING" and notificationTime <= NOW
2. Loops through each pending ticket
3. Sends email via Nodemailer
4. On success: Updates status to "SUCCESS"
5. On failure: Status remains "PENDING" (retried in next cron run)

**Why Every 2 Minutes?**
- Provides timely delivery without excessive database queries
- Balances between responsiveness and resource usage
- Can be adjusted based on requirements

---

### 9. **Database Migration** (`src/migrations/`)

Sequelize migration file creates the database schema:

**Migration Features:**
- **Up**: Creates `NotificationTickets` table with all columns
- **Down**: Drops table (for rollback)
- Uses Sequelize data types (INTEGER, STRING, ENUM, DATE)
- Enforces NOT NULL constraints on required fields
- Sets default status to "PENDING"

**Run migrations:**
```bash
npx sequelize-cli db:migrate
```

---

## ⚙️ Configuration & Setup

### Prerequisites
- Node.js (v14+)
- MySQL 5.7+
- RabbitMQ
- Gmail account with App Password enabled

### Installation Steps

#### 1. Clone Repository
```bash
git clone <repository-url>
cd ReminderService
```

#### 2. Install Dependencies
```bash
npm install
```

Dependencies installed:
- **express**: Web framework
- **body-parser**: Middleware for JSON parsing
- **sequelize**: ORM for MySQL
- **mysql2**: MySQL database driver
- **nodemailer**: Email sending library
- **amqplib**: RabbitMQ client
- **node-cron**: Task scheduling
- **dotenv**: Environment variable loader
- **nodemon**: Auto-restart during development

#### 3. Configure Environment Variables (`.env`)
```env
PORT=3003
EMAIL_ID=your_email@gmail.com
EMAIL_PASS=your_app_password
MESSAGE_BROKER_URL=amqp://localhost:5672
EXCHANGE_NAME=booking_exchange
REMINDER_BINDING_KEY=booking.reminder
```

**Important: Gmail App Password**
- Go to Google Account Security settings
- Enable 2-Factor Authentication
- Generate an "App Password"
- Use the 16-character password (spaces removed) in EMAIL_PASS

#### 4. Configure Database (`src/config/config.json`)
```json
{
  "development": {
    "username": "root",
    "password": "your_mysql_password",
    "database": "flights_REMINDER_DB_",
    "host": "127.0.0.1",
    "dialect": "mysql"
  }
}
```

#### 5. Create Database
```bash
mysql -u root -p
CREATE DATABASE flights_REMINDER_DB_;
exit;
```

#### 6. Run Migrations
```bash
npx sequelize-cli db:migrate
```

#### 7. Start RabbitMQ
- On Docker: `docker run -d --name rabbitmq -p 5672:15672 rabbitmq:3.9-management`
- Or: `rabbitmq-server`

#### 8. Start Application
```bash
npm start
```

You should see:
```
Server started at port 3003
```

---

## 🚀 Usage Examples

### Create Reminder via HTTP API

```bash
curl -X POST http://localhost:3003/api/v1/tickets \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Flight Departure Reminder",
    "content": "Your flight BA123 departs tomorrow at 10:00 AM from London Heathrow",
    "recepientEmail": "passenger@example.com",
    "notificationTime": "2024-03-31T09:00:00Z"
  }'
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "subject": "Flight Departure Reminder",
    "content": "Your flight BA123 departs tomorrow...",
    "recepientEmail": "passenger@example.com",
    "status": "PENDING",
    "notificationTime": "2024-03-31T09:00:00.000Z",
    "updatedAt": "2024-03-31T06:15:00.000Z",
    "createdAt": "2024-03-31T06:15:00.000Z"
  },
  "err": {},
  "message": "Successfully registered an email reminder"
}
```

### Create Reminder via RabbitMQ Message

From another service, publish a message:

```javascript
// Pseudo-code from booking service
channel.publish(
  'booking_exchange',
  'booking.reminder',
  Buffer.from(JSON.stringify({
    service: 'CREATE_TICKET',
    data: {
      subject: 'Booking Confirmation',
      content: 'Your booking is confirmed',
      recepientEmail: 'user@example.com',
      notificationTime: new Date(Date.now() + 86400000) // 24 hours from now
    }
  }))
);
```

The Reminder Service automatically creates this ticket and sends email at the scheduled time.

---

## 🔄 Complete Request-Response Flow Example

### Scenario: Hotel sends booking reminder

**Timeline:**
- T=0: Hotel service publishes RabbitMQ message
- T=0+1s: Reminder Service receives and stores in database
- T=0+24h: Cron job finds PENDING reminder ready to send
- T=0+24h+5s: Email sent via Gmail SMTP
- T=0+24h+10s: Database status updated to "SUCCESS"

**Step-by-step:**

```
1. Hotel Service publishes:
   └─> Exchange: booking_exchange
       RoutingKey: booking.reminder
       Payload: { service: 'CREATE_TICKET', data: {...} }

2. Reminder Service Message Queue receives:
   └─> Parses JSON
   └─> Calls subscribeEvents(payload)

3. subscribeEvents routes to createNotification:
   └─> Calls email-service.createNotification()
   └─> Calls ticket-repository.create()
   └─> Inserts into DB with status=PENDING
   └─> Returns created ticket

4. Every 2 minutes, cron job runs:
   └─> Calls email-service.fetchPendingEmails()
   └─> Query: WHERE status='PENDING' AND notificationTime <= NOW()
   └─> Returns matching tickets

5. For each pending ticket:
   └─> Calls sendBasicEmail(from, to, subject, content)
   └─> Nodemailer connects to Gmail SMTP
   └─> Sends email message
   └─> On success: Calls updateTicket(id, {status: 'SUCCESS'})

6. Database updated:
   └─> Ticket status changed to 'SUCCESS'
   └─> updatedAt timestamp updated
```

---

## 📊 Database Schema Details

### NotificationTickets Table

```sql
CREATE TABLE `NotificationTickets` (
  `id` int NOT NULL AUTO_INCREMENT,
  `subject` varchar(255) NOT NULL,
  `content` varchar(255) NOT NULL,
  `recepientEmail` varchar(255) NOT NULL,
  `status` enum('PENDING','SUCCESS','FAILED') DEFAULT 'PENDING',
  `notificationTime` datetime NOT NULL,
  `createdAt` datetime NOT NULL,
  `updatedAt` datetime NOT NULL,
  PRIMARY KEY (`id`)
);
```

**Indexes (Recommended):**
```sql
CREATE INDEX idx_status ON NotificationTickets(status);
CREATE INDEX idx_notificationTime ON NotificationTickets(notificationTime);
CREATE INDEX idx_status_time ON NotificationTickets(status, notificationTime);
```

---

## 🐛 Troubleshooting

### Issue: "Cannot find module 'sequelize'"
**Solution:** Run `npm install`

### Issue: "ECONNREFUSED" on port 5672 (RabbitMQ)
**Solution:** Ensure RabbitMQ is running. Start with Docker: `docker run -d --name rabbitmq -p 5672:15672 rabbitmq:3.9-management`

### Issue: "Connection refused 127.0.0.1:3306" (MySQL)
**Solution:** Start MySQL service or verify credentials in `config.json`

### Issue: "Invalid login" from Nodemailer
**Solution:** 
- Verify Gmail App Password (not regular password)
- Check EMAIL_ID and EMAIL_PASS in `.env`
- Ensure "Less secure app access" is handled via App Passwords

### Issue: Emails not sending but database shows PENDING
**Solution:** 
- Check `job.js` is uncommented in `index.js`: `jobs();`
- Verify cron string format `*/2 * * * *`
- Check email logs in console for SMTP errors

### Issue: "Invalid binding key"
**Solution:** Ensure `REMINDER_BINDING_KEY` in `.env` matches publisher's routing key

---

## 📈 Performance & Scalability

### Current Limitations & Improvements

| Aspect | Current | Improvement |
|--------|---------|-------------|
| **Email Queue** | Synchronous sending in cron | Use Bull/Bee-Queue for async jobs |
| **Database Queries** | No indexes on status/time | Add composite indexes |
| **Error Handling** | Failed emails stay PENDING | Implement retry logic with exponential backoff |
| **Log Storage** | Console only | Use Winston/Bunyan for persistent logs |
| **Monitoring** | None | Add Prometheus metrics |
| **Testing** | None | Add Jest unit tests |

### Scaling Recommendations

1. **Horizontal Scaling:**
   - Run multiple instances with load balancer
   - Share MySQL and RabbitMQ connection pools

2. **Database Optimization:**
   - Add indexes on (status, notificationTime)
   - Archive old SUCCESS records
   - Use database connection pooling

3. **Message Queue:**
   - Split emails by priority
   - Use different queues for bulk vs urgent notifications

4. **Email Service:**
   - Use SendGrid/AWS SES for higher throughput
   - Implement retry logic with exponential backoff
   - Add email rate limiting

---

## 📝 Common Modifications

### Change Cron Schedule

In `src/utils/job.js`, modify the cron expression:

```javascript
// Every 5 minutes
cron.schedule('*/5 * * * *', async () => { ... });

// Every hour
cron.schedule('0 * * * *', async () => { ... });

// Every day at 2 AM
cron.schedule('0 2 * * *', async () => { ... });
```

### Change Email Provider

Replace Nodemailer with SendGrid in `src/config/emailConfig.js`:

```javascript
const sgMail = require('@sendgrid/mail');
sgMail.setApiKey(process.env.SENDGRID_API_KEY);

const sender = {
  sendMail: async (options) => {
    return sgMail.send({
      to: options.to,
      from: process.env.EMAIL_ID,
      subject: options.subject,
      text: options.text
    });
  }
};

module.exports = sender;
```

### Add Email HTML Template

In `src/services/email-service.js`:

```javascript
const sendBasicEmail = async (mailFrom, mailTo, mailSubject, mailBody) => {
    const response = await sender.sendMail({
        from: mailFrom,
        to: mailTo,
        subject: mailSubject,
        html: `<h1>${mailSubject}</h1><p>${mailBody}</p>`,
        text: mailBody
    });
};
```

---

## 📚 Technology Stack

| Technology | Purpose | Version |
|-----------|---------|---------|
| **Node.js** | Runtime | ^14.0.0 |
| **Express.js** | Web Framework | ^4.18.2 |
| **Sequelize** | ORM | ^6.28.0 |
| **MySQL** | Database | ^2.3.3 |
| **Nodemailer** | Email Service | ^6.8.0 |
| **RabbitMQ (amqplib)** | Message Broker | ^0.10.3 |
| **node-cron** | Job Scheduler | ^3.0.2 |
| **dotenv** | Config Management | ^16.0.3 |
| **body-parser** | Request Parsing | ^1.20.1 |
| **nodemon** | Development | ^2.0.20 |

---

## 📞 Support & Contact

For issues or questions:
1. Check the Troubleshooting section above
2. Review logs in console output
3. Verify all environment variables in `.env`
4. Check RabbitMQ and MySQL are running

---

## 📄 License

ISC

---

## ✅ Summary

This Reminder Service is a complete, production-ready microservice for managing email notifications. It:

✅ Accepts reminder creation via REST API or RabbitMQ  
✅ Persists reminders in MySQL database  
✅ Automatically sends emails at scheduled times  
✅ Tracks email delivery status  
✅ Integrates seamlessly with other microservices  
✅ Scales horizontally with proper configuration  
✅ Handles both transactional and event-driven workflows

Perfect for building notification features in larger applications!
