# Wiring Forward Email into an app

Keep credentials in the environment:

```bash
# .env (gitignored)
SMTP_HOST=smtp.forwardemail.net
SMTP_PORT=465
SMTP_USER=hello@example.com       # the alias
SMTP_PASS=...                     # from generate-password
MAIL_FROM="Example <hello@example.com>"
# only for the HTTP API:
FORWARD_EMAIL_API_KEY=...
```

SMTP ports: 465 or 2465 use implicit TLS (recommended). 587, 2587 or 2525 use STARTTLS.

## Node.js (Nodemailer) — Express, Next.js route handlers, Remix, NestJS, and similar

```js
import nodemailer from 'nodemailer';

const port = Number(process.env.SMTP_PORT || 465);

export const mailer = nodemailer.createTransport({
  host: process.env.SMTP_HOST,           // smtp.forwardemail.net
  port,
  secure: port === 465 || port === 2465, // implicit TLS; 587, 2587 and 2525 upgrade with STARTTLS
  auth: { user: process.env.SMTP_USER, pass: process.env.SMTP_PASS }
});

await mailer.sendMail({ from: process.env.MAIL_FROM, to, subject, text, html });
```

In Next.js, run this only on the server (route handlers, server actions), never in client components.

## HTTP API (any language, no SMTP library)

`POST https://api.forwardemail.net/v1/emails` takes a JSON body of Nodemailer message fields. Authenticate with the API key. `from` must be an alias on the verified domain.

```js
const res = await fetch('https://api.forwardemail.net/v1/emails', {
  method: 'POST',
  headers: {
    // btoa works in Node 16+, Deno, Bun and edge runtimes
    Authorization: 'Basic ' + btoa(`${process.env.FORWARD_EMAIL_API_KEY}:`),
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ from: 'hello@example.com', to: 'user@example.org', subject: 'Welcome', html: '<p>Hi!</p>' })
});
if (!res.ok) throw new Error(`Forward Email ${res.status}: ${await res.text()}`);
```

```bash
curl -X POST https://api.forwardemail.net/v1/emails -u "$FORWARD_EMAIL_API_KEY:" \
  -H "Content-Type: application/json" \
  -d '{"from":"hello@example.com","to":"user@example.org","subject":"Hello","text":"It works"}'
```

The endpoint also accepts alias credentials in place of the API key (`-u "hello@example.com:$SMTP_PASS"`), but only when the alias has `has_imap: true`. The API key is simpler.

## Python (smtplib)

```python
import os, smtplib, ssl
from email.message import EmailMessage

msg = EmailMessage()
msg["From"], msg["To"], msg["Subject"] = os.environ["SMTP_USER"], "user@example.org", "Hello"
msg.set_content("It works")
with smtplib.SMTP_SSL("smtp.forwardemail.net", 465, context=ssl.create_default_context()) as s:
    s.login(os.environ["SMTP_USER"], os.environ["SMTP_PASS"])
    s.send_message(msg)
```

## Django (settings.py)

```python
import os

EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "smtp.forwardemail.net"
EMAIL_PORT = 465
EMAIL_USE_SSL = True
EMAIL_HOST_USER = os.environ["SMTP_USER"]
EMAIL_HOST_PASSWORD = os.environ["SMTP_PASS"]
DEFAULT_FROM_EMAIL = os.environ.get("MAIL_FROM", EMAIL_HOST_USER)
```

## Ruby on Rails (config/environments/production.rb)

```ruby
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address: "smtp.forwardemail.net",
  port: 465,
  user_name: ENV["SMTP_USER"],
  password: ENV["SMTP_PASS"],
  authentication: :plain,
  tls: true
}
```

## Laravel (.env)

```bash
MAIL_MAILER=smtp
MAIL_HOST=smtp.forwardemail.net
MAIL_PORT=587          # STARTTLS is negotiated automatically
MAIL_USERNAME=hello@example.com
MAIL_PASSWORD=...
MAIL_FROM_ADDRESS=hello@example.com
```

## Go (net/smtp, STARTTLS on 587)

```go
auth := smtp.PlainAuth("", os.Getenv("SMTP_USER"), os.Getenv("SMTP_PASS"), "smtp.forwardemail.net")
msg := []byte("From: hello@example.com\r\nTo: user@example.org\r\nSubject: Hello\r\n\r\nIt works\r\n")
err := smtp.SendMail("smtp.forwardemail.net:587", auth, "hello@example.com", []string{"user@example.org"}, msg)
```

## After wiring

Send one real message through the app's own code path to the user's address. Then confirm it with `listEmails` (`GET /v1/emails`) and check the daily limit with `getEmailLimit`.
