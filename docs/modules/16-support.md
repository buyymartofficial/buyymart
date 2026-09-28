# Support and grievance

**Owns:** tickets linked to an order, the visible ticket number, and the clocks for acknowledgement and resolution.  
**Service:** `support-service`.  
**Does not own:** the refund. Support requests a refund. Finance confirms it in payments. Trust handles KYC and suspension.

The 2020 e-commerce rules require a grievance officer on the platform, an acknowledgement within 48 hours, and a resolution within one month. The customer must be able to track the complaint. BuyyMart uses the ticket number for that.

---

## Ticket

| Field | Meaning |
| --- | --- |
| `id` | Shown to the customer as the complaint number |
| `orderId` | Required in version 1. A ticket is about an order. |
| `customerId` | The buyer |
| `sellerId` | The group the complaint is about, when it is about one seller |
| `status` | `open`, `acknowledged`, `waiting_on_customer`, `resolved`, `closed` |
| `openedAt`, `acknowledgedAt`, `resolvedAt` | The clocks |
| `channel` | `app` or `email` |
| `nchReference` | Empty until National Consumer Helpline convergence is switched on |

Messages are a list: author role, body, time. The customer sees staff and their own messages. They do not see internal notes. Internal notes are a flag on the message.

`TicketOpened` is published when the row is created.

---

## Clocks

| Clock | Rule |
| --- | --- |
| Acknowledge | 48 hours from `openedAt`. The first staff reply sets `acknowledged`. The home queue lists tickets that will breach in the next four hours. |
| Resolve | One month from `openedAt`. `resolved` means the customer was told the outcome: refund initiated, return rejected with reason, replacement not offered, or information given. |
| Copy of the complaint | The 2026 amendment briefings say the officer must give the complainant a copy of the complaint as recorded. The ticket page has “Download complaint”, which is the opening message plus the ticket number and time. Ship this whether or not the amendment has commenced. It is useful either way. |

A resolved ticket can be reopened once by the customer within seven days. That does not reset the original one-month clock. Staff see both dates.

---

## Who does what

| Role | Can |
| --- | --- |
| Customer | Open a ticket on their order, reply, download the complaint |
| Seller | Reply on tickets for their groups. They cannot close a complaint. Staff close it. |
| Support | Reply, add internal notes, request a refund |
| Finance | Confirm the refund |
| Admin | Edit the grievance officer name, email, and phone shown in the footer |

The footer of the storefront and the marketing site shows the grievance officer’s name, designation, email, and phone, plus the legal name and address of the company. Seller-specific grievance contacts appear on that seller’s section of the product page.

---

## National Consumer Helpline

Briefings on the September 2026 amendment say participation in the National Consumer Helpline convergence process becomes mandatory, where the earlier rule was a best-effort duty. Confirm the commencement date in the gazette. The ticket already has `nchReference` so a helpline id can be stored without a schema change. An automated feed to the helpline is not part of version 1. Staff can paste the reference.

---

## What support must not do

- Browse customers without an order id, phone, AWB, payment id, or ticket id.
- Mark an order paid.
- Change a price or a GST rate.
- See a card number. There is none to see.
