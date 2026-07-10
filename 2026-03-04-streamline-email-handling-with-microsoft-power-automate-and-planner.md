---
title: Streamline Email Handling with Microsoft Power Automate and Planner
slug: streamline-email-handling-with-microsoft-power-automate-and-planner
date_published: 2026-03-04T05:52:21.000Z
date_updated: 2026-03-04T05:56:49.000Z
excerpt: Emails to devops@company.com are auto-assigned via Power Automate to one active engineer in round-robin, creating a Planner task. SharePoint tracks rotation and conversation IDs to prevent duplicates, ensuring clear ownership and timely responses.
---

We have a team distribution list, for example `devops@company.com`. Everyone in the DevOps team receives those emails. That gives visibility, but it does not give ownership. Over time, the manager noticed the same pattern: some emails were answered late, and some were ignored because everyone assumed someone else would handle them.

The manager asked a simple question: “Who handles this email?” There was no deterministic answer.

We already use Microsoft 365 for email, collaboration, and task tracking. The constraint was to stay inside the ecosystem. The goal was not to build a ticketing system. The goal was to enforce single ownership per incoming email thread.

The solution is a Power Automate flow that converts incoming emails into Planner tasks and assigns them sequentially based on a rotation stored in SharePoint.

This write-up explains every step and expression used.

---

## High-Level Architecture

We use:

- Outlook (distribution list)
- Power Automate (orchestration)
- SharePoint (state storage)
- Microsoft Planner (task tracking)

There are three SharePoint lists involved:

1. **DevopsRotation – Engineers List**

    +------------+------------------------+------------+--------+
    | Title      | Email                  | OrderNumber| Active |
    +------------+------------------------+------------+--------+
    | Engineer 1 | engineer1@example.com  | 1          | True   |
    | Engineer 2 | engineer2@example.com  | 2          | True   |
    | Engineer 3 | engineer3@example.com  | 3          | True   |
    | Engineer 4 | engineer4@example.com  | 4          | True   |
    | Engineer 5 | engineer5@example.com  | 5          | True   |
    +------------+------------------------+------------+--------+

SharePoint Lists - DevopsRotation

The OrderNumber determines the rotation sequence, while an engineer marked as Active = True is included in the rotation. If someone is on leave, their Active status should be set to False.

1. **DevopsRotationState – Rotation Pointer**

    +---------+-----------+
    | Title   | LastOrder |
    +---------+-----------+
    | Rotation| 4         |
    +---------+-----------+

SharePoint Lists - DevopsRotationState

The last assigned engineer had an OrderNumber of 4, so the next email in the rotation will be directed to the engineer with OrderNumber 5.

1. **EmailConversations – Deduplication**

    +----------------------------+--------------------------------------+
    | Title                      | ConversationId                       |
    +----------------------------+--------------------------------------+
    | (empty or email subject)   | (ConversationId of email thread)     |
    +----------------------------+--------------------------------------+

SharePoint Lists - EmailConversations

We use ConversationId instead of Subject because replies maintain the same ConversationId, while the subject line may change.

---

## Conceptual Architecture Diagram

                    ┌─────────────────────────┐
                    │   Distribution List     │
                    │   devops@company.com    │
                    └────────────┬────────────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │   Outlook Inbox  │
                       │   (Flow Trigger) │
                       └─────────┬────────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │   Power Automate │
                       │   Orchestration  │
                       └─────────┬────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
            ▼                    ▼                    ▼
    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │ Email            │  │ DevopsRotation   │  │ DevopsRotation   │
    │ Conversations    │  │ (Engineers List) │  │ State            │
    │ (Dedup Check)    │  │ (Active + Order) │  │ (Last Pointer)   │
    └──────────────────┘  └──────────────────┘  └──────────────────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │   Microsoft      │
                       │   Planner        │
                       │   (Task Created  │
                       │    + Assigned)   │
                       └──────────────────┘

## Flow Overview

    [Trigger: New Email]
            |
            v
    [Compose_ConversationId]
            |
            v
    [Get Items: EmailConversations]
            |
            v
    [Compose_Debug]
            |
            v
    [Compose_Length]
            |
            v
    [Condition: Length == 0 ?]
          /       \
        No         Yes
        |           |
      [Stop]        v
               [Get DevopsRotationState]
                       |
                       v
               [Compose_LastOrder]
                       |
                       v
               [Get Active Engineers]
                       |
                       v
               [Compose_MaxOrder]
                       |
                       v
               [Compose_NextOrder]
                       |
                       v
               [Filter_array by OrderNumber]
                       |
                       v
               [Compose_SelectedEmail]
                       |
                       v
               [Get_user_profile_(V2)]
                       |
                       v
               [Compose_UserId]
                       |
                       v
               [Compose_UserAssignments]
                       |
                       v
               [Create Planner Task]
                       |
                       v
               [Delay 5 seconds]
                       |
                       v
               [Update Task Details]
                       |
                       v
               [Update DevopsRotationState]
                       |
                       v
               [Create EmailConversations item]

---

### 1. Trigger – When a New Email Arrives

We start with the flow watching the **DevOps distribution list**. Every incoming email sparks the automation.

- **Connector:** Office 365 Outlook
- **Trigger:** When a new email arrives (V3)
- **Parameters:**
- `To = devops@company.com`
- `FolderPath = Inbox`

Because this is a shared distribution list and you’re part of it, the email lands in your mailbox, our flow is listening there. Every new thread is captured automatically.

### 2. Extract ConversationId

Instead of relying on the email subject (which can change with replies), we grab the **ConversationId**,a unique identifier that persists across the email thread.

- **Action:** Compose
- **Name:**`Compose_ConversationId`
- **Expression:**

    @triggerOutputs()?['body/conversationId']

This ensures that **all replies stay linked** to the same task, preventing duplicates.

### 3. Check for Existing Conversation

Before creating a task, we check if this conversation has already been handled.

- **Action:** Get Items
- **List:** EmailConversations
- **Filter Query:**

    ConversationId eq '@{outputs('Compose_ConversationId')}'
    

- **Top count:** 1

Then we debug and count:

- **Compose_Debug:**`@body('EmailConversations')?['value']`
- **Compose_Length:**`@length(body('EmailConversations')?['value'])`

**Condition:**

- If `Compose_Length > 0` → Stop (this thread already has a task)
- If `Compose_Length = 0` → Continue (new thread, ready for assignment)

This sets the stage: only **fresh email threads** move forward to the rotation assignment, keeping everything clean and deterministic.

### 4. Read Rotation State

To figure out who should get the next email, we check the rotation’s current position. We use **Get Items** on the `DevopsRotationState` list with this filter:

    Title eq 'Rotation'
    

We only need the first item (`Top Count = 1`) because there’s just one rotation record.

Next, we extract the last assigned engineer’s order number so we can determine who comes next:

    @int(first(body('DevopsRotationState')?['value'])?['LastOrder'])
    

This value (`LastOrder`) acts as a pointer in our round-robin rotation, helping the flow assign the next task in sequence.

### 5. Get Active Engineers

Now that we know where the rotation left off, we need the list of engineers who are available. Using **Get Items** on the `DevopsRotation` list, we filter only those marked as active:

    Active eq 1
    

We also order them by `OrderNumber` ascending so the rotation sequence is preserved. This gives us a clean, ordered pool of engineers ready to receive the next task.

### 6. Compute Maximum Order

With the active engineers in hand, we need to know the size of our rotation. Using a **Compose** action, we calculate the total number of active engineers:

    @length(body('DevopsRotation')?['value'])

This `MaxOrder` becomes our ceiling for the round-robin logic. Think of it as the last number on the spinning wheel, once we hit it, we loop back to the start.

### 7. Compute Next Order (Circular Logic)

Time to pick the next engineer. Here we implement circular logic with another **Compose** action:

    @if(
        equals(outputs('Compose_LastOrder'), outputs('Compose_MaxOrder')),
        1,
        add(outputs('Compose_LastOrder'), 1)
    )

- If the last assigned order equals the maximum → start over at 1.
- Otherwise → simply move to the next order.

This ensures a smooth **round-robin rotation**, giving each engineer a fair turn without skipping anyone.

### 8. Select Engineer by OrderNumber

Now that we know the `NextOrder`, it’s time to pick our engineer. We use a **Filter Array** action on the `DevopsRotation` list:

    From: @body('DevopsRotation')?['value']
    Filter Query: @equals(string(item()?['OrderNumber']), string(outputs('Compose_NextOrder')))
    

Then, with a **Compose** action, we extract the selected engineer’s email:

    @first(body('Filter_array'))?['Email']
    

This is the lucky engineer who will own the incoming email thread.

### 9. Get Azure AD User Profile

Next, we fetch the user profile from Azure AD so Planner can recognize them:

- **Action:** Get_user_profile_(V2)
- **Parameter:**`@outputs('Compose_SelectedEmail')`

From here, extract the **User ID** using another **Compose**:

    @body('Get_user_profile_(V2)')?['id']
    

This ID is what Planner needs to assign tasks properly.

### 10. Build Planner Assignment Object

Planner wants a structured JSON for assignments. We create it in a **Compose** action:

    @json(concat(
      '{ "',
      outputs('Compose_UserId'),
      '": { "@odata.type": "#microsoft.graph.plannerAssignment", "orderHint": " !" } }'
    ))
    

This is basically wrapping the engineer’s ID in Planner-friendly armor.

### 11. Create Planner Task

With everything ready, we now create the task in Planner:

- **Title:**`@concat('[PA TEST] ', triggerOutputs()?['body/subject'])`
- **Start Date:**`@triggerOutputs()?['body/receivedDateTime']`
- **Due Date:**

    @formatDateTime(
        addDays(
            convertTimeZone(triggerOutputs()?['body/receivedDateTime'], 'UTC', 'SE Asia Standard Time'),
            7
        ),
        'yyyy-MM-ddTHH:mm:ss'
    )
    

- **Assignments:**`@outputs('Get_user_profile_(V2)')?['body/id']`

This creates a task for exactly one engineer, the one next in rotation.

### 12. Delay

A small pause is needed. We wait **5 seconds** to ensure Planner fully registers the task before we add details.

### 13. Update Task Details

Now that the task exists, we enrich it with the email’s content so the engineer has full context.

- **Task Id:**`@body('Create_a_task')?['id']`
- **Description:**

    @concat(
        'From: ', triggerOutputs()?['body/from'],
        decodeUriComponent('%0A'),
        'Received: ',
        formatDateTime(
            convertTimeZone(triggerOutputs()?['body/receivedDateTime'], 'UTC', 'SE Asia Standard Time'),
            'HH:mm ''WIB'' - dddd, dd MMMM yyyy'
        ),
        decodeUriComponent('%0A%0A'),
        '---',
        decodeUriComponent('%0A%0A'),
        if(
            empty(triggerOutputs()?['body/bodyPreview']),
            'This email is protected or has no preview available.',
            substring(
                coalesce(triggerOutputs()?['body/bodyPreview'], ''),
                0,
                min(5000, length(coalesce(triggerOutputs()?['body/bodyPreview'], '')))
            )
        )
    )
    

This captures who sent the email, when it was received, and a preview of the message. The engineer now has all info in one place.

### 14. Update DevopsRotationState

To keep the rotation moving, we update the “last assigned engineer” pointer:

- **Patch DevopsRotationState Id:**`@first(body('DevopsRotationState')?['value'])?['ID']`
- **LastOrder:**`@outputs('Compose_NextOrder')`

This ensures the next incoming email will go to the next engineer in line, round-robin style.

### 15. Create EmailConversations Item

Finally, we record that this conversation has been handled so we don’t duplicate tasks:

- **Title:**`@triggerOutputs()?['body/subject']`
- **ConversationId:**`@outputs('Compose_ConversationId')`

With this, the flow has full memory of every thread, ensuring **single ownership** per email.

---

### 🎯 Final Notes

- Engineers on leave? Set `Active = 0` in `DevopsRotation` They automatically skip the rotation.
- Concurrency isn’t locked; simultaneous emails could theoretically race, but in practice it’s rare.
- Every new email thread results in **exactly one Planner task**, assigned deterministically.
- When someone asks, “Who handles this email?”, the answer is visible in Planner, clear as day.

This flow is still evolving, but it’s already a **solid framework for predictable ownership** in a team inbox.
