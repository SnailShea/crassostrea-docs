# Journaling

The gateway keeps messages for at most 31 days ([mail log](mail-log.md)). For
long-term archiving it sends a copy of every message to an archive: MailStore,
an Exchange or Microsoft 365 journal mailbox, or any archive that accepts
mail.

## Settings

**Mail log → Journaling**:

| Setting | Default | Meaning |
| --- | --- | --- |
| Archive address | empty (off) | Where the copies go |
| Inbound mail, Outbound mail | both on | Which directions are journaled |
| Also rejected and greylisted mail | off | Normally only delivered mail (and mail marked as spam) |
| Format | Journal report | See below |

**Send a test** sends a journal report of a test message.

## Formats

- **Journal report** (default): a message from `journal@<hostname>` with the
  envelope sender, **every recipient** (Bcc recipients too, which the message
  itself does not show), the Message-ID, direction, time, client and spam
  filter result, and the original message attached unchanged, byte for byte
  (`message/rfc822`). Its header `X-Spamfilter-Journal` says the direction.
- **Copy of the message**: the message itself, sent to the archive address.
  For archives that cannot read journal reports; the envelope (Bcc
  recipients) is lost.

## Delivery

When the mail log stores a message, it is queued in Valkey;
`spamfilter-journal.service` sends the copies through Postfix's local port
without the filter, so a journal copy is neither scanned nor logged again. If
the archive cannot be reached, copies wait in Valkey and are sent later; the
page shows how many wait, how many were sent and the last error. Mail is
never held up by the archive. In a cluster every node runs the service on the
shared queue.

Copies come from `journal@<hostname>`: make the archive accept that sender
from the gateway. Bounces from the archive go to that address, never to the
original sender.

API: settings `journal_*`, `GET /api/v1/journal/status`.
