# vSL stages and SMTP states

vSMTP can interact with the messaging transaction at multiple levels. These are related to the states defined in the SMTP protocol.

At each step vSL updates and publishes a global context containing transaction and mail data.

## vSMTP stages

| Stage   | SMTP state                 | Context available               |
| :------ | :------------------------- | :------------------------------ |
| connect | Before HELO/EHLO command   | Connection related information. |
| helo    | After HELO/EHLO command    | HELO string.                    |
| mail    | After MAIL FROM command    | Sender address.                 |
| rcpt    | After each RCPT TO command | The entire SMTP envelop.        |
| preq    | Before queuing[^preq]      | The entire mail.                |
| postq   | After queuing[^postq]      | The entire mail.                |
| deliver | Before delivering          | The entire mail.                |

[^preq]: Preq stage triggers after the end of data, before the server answer (ex. 250 OK).
[^postq]: Postq stage triggers when Connection is already closed and the SMTP code sent.

## Before queueing vs. after queueing

vSMTP can process mails before the incoming SMTP mail transfer completes and thus rejects inappropriate mails by sending an SMTP error code and closing the connection.

This early detection of inappropriate mails has several advantages :

- The responsibility is on the remote SMTP client side,
- It consumes less CPU and disk resources,
- The system is more reactive.

However as the SMTP transfer must end within a deadline, a system facing a heavy workload may have difficulties to respond in time.

To protect against bursts and crashes, vSMTP implements several internal mechanisms like 'delay-variation' or 'temporary service unavailable messages', in accordance with the SMTP RFCs.

## Context variables

As described above, depending on the stage vSL exposes variables to the end user.

| Stage       | Name                  | Type            | Description                      |
| :---------- | :-------------------- | :-------------- | :------------------------------- |
| Connect     | client_ip             | ip4/ip6         | Source IP address.               |
|             | client_port           | int             | Source port.                     |
|             | connect_timestamp     | POSIX timestamp | Connection timestamp.            |
| Helo        | helo                  | string          | HELO/EHLO SMTP value.            |
| Mail        | mail_from             | addr            | Sender email address.            |
|             | mail_from.local_part  | string          | Sender identifier.               |
|             | mail_from.domain      | fqdn            | Sender fqdn.                     |
| Rcpt        | rcpt                  | addr            | the current recipient addresses. |
|             | rcpt.local_part       | string          | current recipient identifier.    |
|             | rcpt.domain           | fqdn            | current recipient fqdn.          |
| Rcpt        | rcpt_list             | array[addr]     | Array of recipient addresses.    |
|             | rcpt_list.local_parts | array[string]   | Array of recipient identifier.   |
|             | rcpt_list.domains     | array[fqdn]     | Array of recipient fqdn.         |
| Next stages | mail                  | string          | Email raw data.                  |

These variables are part of the email context `ctx`. Thus they must be called in a vSL file using the dot notation i.e. `ctx.timestamp`.

## Connection vs mail transaction

As defined in the SMTP RFCs, a single connection can handle several mail transactions.

```shell
[... connection from an IP]
HELO                                    # Start of SMTP transaction 
    > MAIL FROM > RCPT TO > DATA        # First mail 
    > MAIL FROM > RCPT TO > DATA        # Second mail
    > [...]
QUIT                                    # End of transaction
```

&#9762; | the "mail context" (obtained with the function `ctx()`) is unique for each incoming connexion.
