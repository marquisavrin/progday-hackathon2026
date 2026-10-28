# Fazbear GuestFlow — System Design & Architecture

**Project Theme:** Fazbear 2.0 — Reinvent the Pizzeria  
**Innovation Track:** Guest Experience  
**Project Type:** Hackathon prototype  
**Development Time:** 3 hours

---

## 1. Project Overview

**Fazbear GuestFlow** is a guest queue management prototype designed to make waiting at a busy pizzeria clearer and more convenient. Guests can join a virtual queue, receive a ticket number, and check their queue position and estimated waiting time. Staff can view the queue, call the next guest, and mark guests as served.

The prototype focuses on one complete guest journey instead of trying to implement a full restaurant management platform.

## 2. Problem Statement

Traditional restaurant queues can lead to crowded waiting areas, uncertainty about waiting times, and repeated questions directed at reception staff. Guests may not know their position in line or when they should return to the counter.

GuestFlow addresses this by providing a digital queue and a staff dashboard that makes queue status easier to understand.

## 3. Goals and Objectives

### Goals
- Make the guest queue easier to understand.
- Give guests a ticket number and queue status.
- Display a simple estimated waiting time.
- Let staff manage the queue from a dashboard.
- Demonstrate a realistic, working prototype within three hours.

### Success Criteria
The prototype is successful if a demo user can:
1. Join the queue.
2. Receive a ticket number.
3. View their queue position and estimated wait.
4. See the ticket status change when staff call them.
5. Be marked as served by staff.

## 4. Scope

### Must-have features
- Guest queue registration using a guest name and party size.
- Automatic ticket number generation.
- Queue position and estimated waiting time.
- Staff dashboard showing active queue tickets.
- Staff action to call the next waiting guest.
- Staff action to mark the called guest as served.
- Basic input validation and visible status labels.

### Optional features (only if time permits)
- Refresh-safe persistence.
- Responsive mobile layout.
- Cancel a ticket.
- A simple “Your Turn” notice on the guest ticket screen.
- Real-time synchronization across devices, if using a shared backend.

### Out of scope for the 3-hour prototype
- Online payments and food ordering.
- SMS, email, or push notifications.
- AI-based wait prediction.
- Full table reservation management.
- Customer accounts and complex authentication.
- Production-grade analytics and restaurant integrations.

## 5. Recommended Technology Stack

### Fastest single-device prototype
- **Frontend:** React + TypeScript
- **Styling:** Tailwind CSS or plain CSS
- **State:** React `useState`
- **Persistence:** `localStorage`

This is suitable when the team demonstrates the guest and staff views in the same browser. Browser storage is not a shared database and does not automatically synchronize separate devices.

### Shared multi-device prototype
- **Frontend:** React + TypeScript
- **Styling:** Tailwind CSS or plain CSS
- **Backend and database:** Supabase
- **Live updates:** Supabase Realtime, if configured and tested

Use a shared backend if guests use their phones while staff use a separate laptop. Keep database permissions enabled and appropriately configured; do not expose service-role secrets in frontend code.

**Decision rule:** Check the competition requirements first. If a backend is required, or the demo must share live queue state across devices, use the shared-backend option. Otherwise, build the single-browser version first and prioritize a reliable end-to-end demo.

## 6. System Architecture

### Option A — Single-browser prototype

```text
+---------------------------+
|     Guest Interface       |
| Join Queue / View Ticket  |
+-------------+-------------+
              |
              v
+---------------------------+
|   Queue Management Logic  |
| Ticket IDs / Position     |
| Wait Estimate / Status    |
+-------------+-------------+
              |
              v
+---------------------------+
|      localStorage         |
| Saved tickets in browser  |
+-------------+-------------+
              ^
              |
+-------------+-------------+
|      Staff Dashboard      |
| View / Call / Mark Served |
+---------------------------+
```

In this option, the guest interface and staff dashboard use the same app state and browser storage. It is simple to build, but it is not suitable for independent devices that need shared data.

### Option B — Shared multi-device prototype

```text
+-------------------+       +-------------------+
| Guest Interface   |       | Staff Dashboard   |
| Mobile / Browser  |       | Laptop / Browser  |
+---------+---------+       +---------+---------+
          |                           |
          +-------------+-------------+
                        |
                        v
             +----------------------+
             | Backend / API        |
             | Validate operations  |
             | Manage queue state   |
             +----------+-----------+
                        |
                        v
             +----------------------+
             | Shared Database      |
             | Queue tickets        |
             +----------+-----------+
                        |
                        v
             +----------------------+
             | Realtime Updates     |
             | Notify connected UIs |
             +----------------------+
```

For a short hackathon, Supabase can provide a hosted database and realtime capabilities without requiring the team to build a custom server. Queue-changing operations still need sensible validation and database permissions.

## 7. Main Components

### Guest Interface
- Collects guest name and party size.
- Submits a request to join the queue.
- Displays ticket number, status, queue position, and estimated wait.
- Shows a clear message when the guest is called.

### Queue Management Logic
- Creates tickets with unique IDs.
- Assigns sequential display numbers.
- Determines the number of waiting parties ahead.
- Estimates waiting time using a configurable demo assumption.
- Enforces valid status transitions.

### Staff Dashboard
- Displays active queue tickets.
- Shows guest name, party size, ticket number, and status.
- Allows staff to call the next waiting guest.
- Allows staff to mark a called guest as served.

### Data Storage
- Stores ticket records.
- For the single-browser version, uses `localStorage`.
- For the shared version, uses a database accessed through the backend service.

## 8. Data Model

The prototype can begin with a single `QueueTicket` model.

```ts
export type QueueStatus =
  | "waiting"
  | "called"
  | "served"
  | "cancelled";

export type QueueTicket = {
  id: string;
  ticketNumber: string;
  guestName: string;
  partySize: number;
  status: QueueStatus;
  createdAt: string;
  calledAt?: string;
  servedAt?: string;
};
```

### Field descriptions

| Field | Purpose |
|---|---|
| `id` | Unique internal identifier for a ticket |
| `ticketNumber` | Human-readable ticket, such as `A-012` |
| `guestName` | Name entered by the guest |
| `partySize` | Number of people in the party |
| `status` | Current queue status |
| `createdAt` | Time the ticket was created |
| `calledAt` | Optional time the guest was called |
| `servedAt` | Optional time the guest was served |

For a Supabase version, this can map to a `queue_tickets` table with a UUID primary key, a unique ticket number, validated party size, status, and timestamps. The backend or database must prevent duplicate ticket numbers and conflicting queue actions.

## 9. Queue Status Lifecycle

```text
          +-----------+
          |  Waiting  |
          +-----+-----+
                |
                | Staff calls next
                v
          +-----------+
          |  Called   |
          +-----+-----+
                |
                | Staff marks served
                v
          +-----------+
          |  Served   |
          +-----------+

Waiting -- guest/staff cancels --> Cancelled
```

Rules:
- Only a `waiting` ticket can be called next.
- Only a `called` ticket can be marked as served.
- A served or cancelled ticket must not be called again.
- The next guest is the earliest eligible waiting ticket, unless the team deliberately implements another ordering rule.
- In a shared backend, queue transitions should be validated on the server/database side to reduce race conditions.

## 10. Queue Logic and Wait-Time Estimate

### Queue position
Count the number of active `waiting` tickets ahead of the guest. Exclude served and cancelled tickets.

### Estimated wait
For a simple demonstration, use an assumed average service time:

```text
Estimated wait (minutes) =
    Number of waiting parties ahead × Assumed minutes per party
```

Example: if four waiting parties are ahead and the assumed service time is five minutes per party, the displayed estimate is 20 minutes.

This is only a demo estimate. Real waiting times depend on table availability, party size, staffing, and service speed. Label the estimate as approximate.

### Ticket number
Use a sequential display number such as `A-001`, `A-002`, and `A-003`. In a multi-device version, ticket-number assignment must be centralized or protected by a database transaction/constraint to prevent duplicates.

## 11. Main User Flow

1. Guest opens GuestFlow.
2. Guest enters a name and party size.
3. System validates the input and creates a ticket.
4. Guest sees the ticket number, queue position, and estimated wait.
5. Staff views the ticket on the dashboard.
6. Staff clicks **Call Next Guest**.
7. The earliest waiting ticket changes to `called`.
8. The guest ticket displays **Your Turn** when the status is updated.
9. Staff marks the ticket as served.
10. The ticket is removed from the active waiting list or shown in a served section.

## 12. Suggested UI Screens

### Screen A — Guest Join Queue
- Fazbear GuestFlow title and short explanation.
- Guest name field.
- Party-size selector.
- “Join Queue” button.
- Current queue/wait estimate, if available.

### Screen B — Guest Ticket
- Large ticket number.
- Current status badge.
- Queue position.
- Approximate wait time.
- “Your Turn” message when called.

### Screen C — Staff Dashboard
- Number of waiting parties.
- List/table of tickets.
- Guest name, party size, ticket number, and status.
- “Call Next Guest” button.
- “Mark Served” button for the called ticket.

## 13. Error Handling and Basic Validation

- Guest name must not be empty.
- Party size must be a positive integer within a reasonable limit.
- Disable or guard actions while an operation is in progress.
- Show a message if the queue is empty.
- Prevent calling the same ticket twice.
- Handle storage or network failures with a visible error message.
- Do not silently overwrite tickets when saving.
- For a shared backend, configure access policies so guests cannot arbitrarily mark other guests as served or alter staff-only operations.

## 14. Three-Hour Implementation Plan

| Time | Task | Expected result |
|---|---|---|
| 0–15 min | Agree on scope, roles, repository, and shared data type | Team setup complete |
| 15–60 min | Build ticket creation, numbering, queue position, and wait estimate | Guest can join the queue |
| 60–105 min | Build staff dashboard and queue actions | Staff can call and serve guests |
| 105–135 min | Connect guest status, persistence, and basic styling | End-to-end flow works |
| 135–165 min | Test the complete workflow and fix critical bugs | Demo is reliable |
| 165–180 min | Rehearse the pitch and prepare demo data | Presentation ready |

If using Supabase, set up the table and permissions early, and test a guest action from one browser and a staff action from another. Do not wait until the last 30 minutes to discover that the two interfaces do not share state.

## 15. Team Responsibilities

For a team of four:

1. **Queue logic:** ticket model, numbering, queue position, wait calculation, and status rules.
2. **Guest interface:** join form, ticket view, and status display.
3. **Staff dashboard:** queue list, Call Next, and Mark Served.
4. **Integration and demo:** repository setup, shared types, integration, testing, and presentation.

For a team of three, combine integration and presentation duties with the other roles.

Agree on component props, function names, and the `QueueTicket` type before splitting the work. Integrate early and keep the main branch runnable.

## 16. Demo Acceptance Checklist

- [ ] A guest can join the queue.
- [ ] A valid ticket number is generated.
- [ ] Queue position is displayed correctly.
- [ ] Estimated wait is shown as an approximation.
- [ ] Staff can call the next waiting guest.
- [ ] Guest status changes to `called` / “Your Turn.”
- [ ] Staff can mark the called guest as served.
- [ ] Served tickets are not called again.
- [ ] Refreshing the page does not unexpectedly erase the queue, if persistence is in scope.
- [ ] If using a shared backend, guest and staff views stay consistent across separate devices.
- [ ] The application can be started reliably for the demo.

## 17. Expected Impact

GuestFlow aims to:
- Reduce uncertainty while guests wait.
- Reduce crowding around the reception desk.
- Help guests understand their place in the queue.
- Give staff a clearer view of waiting guests.
- Improve communication between guests and staff.

These are intended benefits; the prototype does not yet prove measured reductions in waiting time or congestion.

## 18. Future Improvements

If developed beyond the hackathon, potential additions include:
- Table reservations and check-in.
- Accessibility settings and multilingual support.
- SMS or push notifications.
- More accurate wait estimates based on historical service data.
- Staff authentication and role-based permissions.
- Queue analytics and peak-hour reporting.
- Integration with table availability and restaurant ordering systems.

## 19. Final Architecture Decision

**Recommended first step:** Build a polished, end-to-end single-browser prototype unless the competition requires a backend or the judges expect guests and staff to use different devices.

**Use a shared backend** if cross-device synchronization is part of the required demo. In that case, use React + TypeScript with Supabase, configure database access correctly, and keep the feature scope limited to the queue workflow.

The key deliverable is one reliable journey:

**Join queue → Receive ticket → Track wait → Get called → Get served.**
