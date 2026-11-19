# fpf-yimby

Filter the daily New North End Front Porch Forum digest with an LLM and email myself only the parts
I care about (mainly events), instead of the full digest going to junk.

## Architecture

```
FPF ──▶ SES inbound ──▶ S3 (raw .eml) ──▶ Lambda ──▶ Bedrock (Claude Haiku) ──▶ SES send ──▶ Gmail
                                            │
                                            └──▶ DynamoDB (already-sent items)
```

- **SES inbound**: receives the digest at `digest@fpf.<domain>`; a receipt rule writes it to S3.
- **S3 → Lambda** (Python):
  1. Parse the MIME message and convert the HTML part to text (the text/plain part is only a stub
     unless FPF's "text only" setting is enabled). `fixtures/sample-digest.eml` is a synthetic sample.
  2. Call Bedrock (Haiku) and get structured JSON back:
     - keep: events (title, date/time, location, link), plus the "Community Calendar" block
     - drop: Discussion threads and replies, paid ads
  3. Dedup: drop items already sent (kept or filtered), keyed on a hash of normalized title (lowercased, whitespace
     collapsed) + author, stored in DynamoDB with a ~30-day TTL. Exact matching is enough since
     people repost with the same title.
  4. Render a short HTML email (events first, sorted by date) and send it with SES. If nothing new
     is left, skip sending.
     - The model also returns the titles it dropped. A **Filtered** section at the bottom lists
       them, to catch mistakes. Filtered titles go through the same dedup and are recorded in
       DynamoDB too, so each appears once. Skip sending only when both sections are empty.
- **Delivery**: SES sends from `digest@fpf.<domain>` to my Gmail. Add a Gmail filter on that sender:
  "Never send to Spam".
- **Failure handling**: the raw email stays in S3, so a failed run can be reprocessed.
- **IaC**: CDK stack (SES receipt rule, S3 bucket, Lambda, DynamoDB table, IAM). DNS records are
  added by hand.

Cost is negligible: one small Bedrock call and two SES messages per day.

## Setup

### 1. SES domain identity (us-east-1)

Use a subdomain (`fpf.<domain>`) so the MX record doesn't take over the root domain's existing mail.

SES console → Identities → Create identity → Domain → `fpf.<domain>`, Easy DKIM on. SES lists 3 CNAMEs.

### 2. DreamHost DNS

Panel → Websites → DNS for the domain → Add Record. Names are relative to the domain.

| Type     | Name                     | Value                                              |
|----------|--------------------------|----------------------------------------------------|
| CNAME ×3 | `<token>._domainkey.fpf` | `<token>.dkim.amazonses.com` (from SES)            |
| MX       | `fpf`                    | `inbound-smtp.us-east-1.amazonaws.com`, priority 10 |

Unconfirmed: whether DreamHost allows a custom MX on a subdomain. If not, delegate `fpf` to a
Route 53 hosted zone via NS records and add the records there.

Wait for the SES identity to show **Verified**.

### 3. SES sandbox

New SES accounts can only send to verified addresses. Verify my Gmail address as an identity
(confirmation link); that's enough for sending to myself, so production access isn't needed.

### 4. Bedrock

Enable model access for Claude Haiku in us-east-1.

### 5. Deploy and switch over

1. Deploy the CDK stack.
2. Send a test email to `digest@fpf.<domain>` and confirm it lands in S3 and the Lambda runs.
3. Change the FPF subscription email (My Account → Manage My Subscription) to
   `digest@fpf.<domain>`.

## Testing

E2E test in this repo, using `fixtures/sample-digest.eml` as the fixture: a fictional issue
(No. 6011, Dec 1, 2026) in FPF's HTML layout, with made-up neighbors and posts.

1. Run the Lambda's processing on the fixture: parse → **real Bedrock call** → dedup → render.
   SES send and DynamoDB are stubbed (empty table, capture the outgoing email).
2. Assert on which items were kept, not on exact wording (model output varies run to run):
   - kept:
     - Holiday Lantern Walk at Leddy Park, Dec 5 (also listed in the Community Calendar)
     - Free Kids' Bike Repair Clinic, Sat Dec 5 (an event inside a business Announcement)
     - Winter Concert at the North End Library, Dec 10
     - Holiday Craft Fair at the Senior Center, Dec 12
     - Calendar only: Cookie Swap (Dec 6), Board Game Night (Dec 9)
   - dropped: both "Re: North Ave Bike Lane Proposal" replies, the Plattsburg Ave apartments
     post (a Discussion that mentions a public hearing date), "Dogs Off Leash at the Beach",
     both paid ads. These titles appear in the email's Filtered section.
   - not asserted until the "what else to keep" question is settled: lost cat, free couch, snow
     tires, chimney sweep request, parking ban reminder.
3. Run it twice against the same stubbed table and assert the second run sends no email at all
   (kept and filtered items are both deduped).
4. Pre-seed the table with the key of an item the model drops, and assert it's absent from the
   Filtered section.

Needs AWS credentials with Bedrock access, so it runs locally rather than in unauthenticated CI.

## Open questions

- Which domain to use.
- What to keep besides events (free items? lost pets? recommendations?).
