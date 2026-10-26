---
id: email-sender-tool
title: Email Toolkit
sidebar_label: Email Toolkit
pagination_prev: user-guide/tools_integrations/tools/overview
pagination_next: null
sidebar_position: 14
---

<!-- cspell:words UIDVALIDITY UIDPLUS -->

# Email Toolkit

The Email toolkit lets assistants and workflows read a mailbox through IMAP and send messages through SMTP. You can enable receiving, sending, or both in one Email integration, then select only the tools your assistant or workflow needs.

This page describes the unified **Email** toolkit. Older deployments may still show the SMTP-only **Notification → Email** tool. See [Upgrading from the Email Sender tool](#upgrading-from-the-email-sender-tool) for the changes.

## Configure an Email integration

Create an integration with credential type **Email**, select its project, and give it an alias. Enable at least one capability:

- **Receive emails (IMAP)** for listing, reading, copying, moving, or deleting messages.
- **Send emails (SMTP)** for sending messages.

### Mailbox and server settings

| Setting           | Credential key       | When to provide it                                                            |
| ----------------- | -------------------- | ----------------------------------------------------------------------------- |
| Receive emails    | `inbound_enabled`    | Set explicitly to `true` or `false`.                                          |
| Send emails       | `outbound_enabled`   | Set explicitly to `true` or `false`. At least one capability must be enabled. |
| IMAP server       | `imap_url`           | For receiving. Use `host:port`; the default is `outlook.office365.com:993`.   |
| Mailbox to access | `imap_email_address` | Required when receiving is enabled.                                           |
| SMTP server       | `smtp_url`           | For sending. Use `host:port`; the default is `smtp.office365.com:587`.        |
| Sender address    | `smtp_from_email`    | Required when sending is enabled.                                             |

The login identity, mailbox address, and sender address can differ. For example, a service account can access a shared mailbox and send from another address when the mail server grants the necessary permissions. A sender override in a tool call does not grant permission to send as that address.

### Authentication

Choose one authentication method for the enabled capabilities:

| Method                                                      | Credential keys                                             | Requirements                                                                                                                |
| ----------------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Basic (`auth_type=basic`)                                   | `username`, `password`                                      | Both are required. Use an app password when your provider requires one and your organization permits it.                    |
| Microsoft Entra application OAuth (`auth_type=oauth_azure`) | `oauth_tenant_id`, `oauth_client_id`, `oauth_client_secret` | All three are required. An administrator must configure the application's access to the mailbox and enabled mail protocols. |

For Microsoft Entra authentication, `oauth_authority` defaults to `https://login.microsoftonline.com` and `oauth_scope` defaults to `https://outlook.office365.com/.default`. Basic authentication is the default method.

Your mail provider must allow the selected authentication method. If organization policy disables Basic authentication or app passwords, use an authentication method approved by your administrator.

Use **Test Integration** before saving or using the integration. The test checks each enabled capability: IMAP connects and selects `Inbox`; SMTP connects, authenticates, and checks the connection without sending a message. A successful test does not prove that every folder or sender override is permitted.

### TLS and private certificate authorities

IMAP uses TLS, and SMTP uses STARTTLS. CodeMie verifies the server certificate and hostname before authenticating.

:::note Private or corporate mail servers
If your mail server uses a certificate signed by a private certificate authority (CA), ask your administrator to add that CA to the CodeMie runtime's trust store. An untrusted, expired, or incorrectly named certificate causes the connection to fail.
:::

## Enable tools in an assistant or workflow

In the assistant's tool selection, choose **Email** and add the tools you need. Select your integration or use the integration lookup configured for the assistant. For a workflow Tool state, select the Email tool and the intended integration alias.

Enabling IMAP does not automatically expose every Email tool to an assistant. Give a reader only the read tools it needs; add sending or mailbox modification tools only when required by the task.

| Tool                    | Purpose                                                 | Main arguments                                   |
| ----------------------- | ------------------------------------------------------- | ------------------------------------------------ |
| `list_email_folder`     | List message IDs in pages without modifying the folder. | Optional `folder`, `limit`, and `after_email_id` |
| `read_email`            | Read a message, participants, and attachment metadata.  | `email_id`, optional `convert_body_to_markdown`  |
| `read_email_attachment` | Retrieve supported attachment content.                  | `email_id`, `attachment_id`                      |
| `copy_email`            | Copy a message to an existing folder.                   | `email_id`, `destination_folder`                 |
| `move_email`            | Move a message to an existing folder.                   | `email_id`, `destination_folder`                 |
| `delete_email`          | Permanently delete the selected message.                | `email_id`                                       |
| `send_email`            | Send a message through SMTP.                            | `recipient_emails`, `subject`, `body`            |

### Read messages and attachments

Use the exact `email_id` returned by a listing or another Email tool. IDs include the folder and its IMAP UIDVALIDITY. A move changes the message's location; use the returned `destination_email_id` when available, or list the destination folder to obtain its ID. Copying leaves the source ID valid.

`read_email` converts HTML bodies to Markdown by default. Set `convert_body_to_markdown=false` only when you need the HTML representation. Plain-text bodies are unchanged.

Attachment metadata from `read_email` includes `attachment_id` and `encoded_size_bytes`: the MIME-encoded payload size, or `null` when unavailable. Call `read_email_attachment` for the attachment content and decoded `size_bytes`.

Supported attachments include text files, HTML, PDF, Word (`.docx`), PowerPoint (`.pptx`), Excel, and PNG/JPEG/GIF/WebP images. Text files return text; HTML and supported documents return Markdown. Documents without extractable text return an error.

Limits are:

- **25 MiB per message**, checked before downloading the full message.
- **10 MiB per decoded attachment**, checked before conversion.
- **5 MiB per image**, with PNG, JPEG, GIF, and WebP supported and checked by byte signature.

Oversized or unsupported content returns an error. Assistant calls can pass supported images to the model as image artifacts. Workspace scripts and direct workflow Tool states receive image content as base64, so avoid placing large image results in workflow context when they are not needed.

### List messages with pagination

`list_email_folder` reads `Inbox` when `folder` is omitted and returns IDs in ascending IMAP UID order. `limit` defaults to 1,000 and accepts values from 1 to 1,000.

For the first page, omit `after_email_id`:

```json
{
  "folder": "Inbox",
  "limit": 2
}
```

The result can be:

```json
{
  "folder": "Inbox",
  "email_items": [
    { "email_id": "imap:Inbox:12345:10" },
    { "email_id": "imap:Inbox:12345:11" }
  ],
  "has_more": true,
  "next_email_id": "imap:Inbox:12345:11"
}
```

When `has_more` is `true`, call the tool again with the same folder and pass `next_email_id` as `after_email_id`. The next page contains UIDs strictly greater than the continuation UID. On the final page, `has_more` is `false` and `next_email_id` is `null`.

For the example above, the next call's arguments are:

```json
{
  "folder": "Inbox",
  "limit": 2,
  "after_email_id": "imap:Inbox:12345:11"
}
```

Continue using each page's returned `next_email_id` until `has_more` is `false`. An empty folder returns `email_items: []`, `has_more: false`, and `next_email_id: null`.

You can move or delete processed messages before requesting the next page: the continuation message does not need to remain in the source folder. The continuation must refer to the same folder and UIDVALIDITY. If the mailbox's UIDVALIDITY changes, discard the continuation and start a new listing.

Pages reflect the current mailbox, not a fixed snapshot. Start each new scheduled processing run **without** `after_email_id` so that earlier messages left for retry are not skipped. This continuation is for paging within a run, not a durable processing checkpoint.

### Send messages

Provide `recipient_emails` as a non-empty list, plus `subject` and `body`. The body can contain HTML. Optional arguments are `cc_emails`, `bcc_emails`, `from_email`, and `timeout` (30 seconds by default).

If `from_email` is omitted, the tool uses `smtp_from_email` from the integration. Any override must be authorized by the SMTP server. Sending requires the outbound capability even when the integration also supports receiving.

A sending failure raises a tool error. A direct workflow Tool state can stop the run on that error; do not assume that the next state will execute or that a failure will be returned as a successful text result. Agent-mediated tool error handling follows the agent's execution policy.

For a complete workflow that generates an onboarding email and pauses for approval before sending it, see the [human-in-the-loop workflow example](../../workflows/configuration/examples.md#115-example-human-in-the-loop-workflow-interrupt_before).

### Move, copy, and delete messages

Create destination folders before calling `move_email` or `copy_email`. Moving requires the server's IMAP `MOVE` capability.

:::warning Permanent deletion
`delete_email` requires IMAP `UIDPLUS` and uses UID-scoped expunge. It fails before setting the deletion flag if the server does not support safe deletion. Confirm the intended message before invoking it. Moving to a review or processed folder is preferable when you need an audit trail or recovery.
:::

`delete_email` is unavailable through workspace-script calls. Its description's request for confirmation does not itself enforce a human approval step; configure the applicable tool approval policy or workflow control before allowing destructive actions.

## Untrusted email content and prompt injection

:::warning Treat message content as data
Email bodies, subjects, sender display names, and attachment contents are untrusted input, even when retrieved from an authorized mailbox. They can contain instructions intended to make an assistant forward confidential data, change recipients, or delete messages. A sender display name is not proof of identity. Markdown conversion does not make the content safe or prevent prompt injection.
:::

When building an agent or an unattended workflow:

- Give the reading or classification assistant only the tools necessary for analysis. Separate classification from sending, moving, or deleting messages.
- State in the assistant's system prompt that message and attachment contents are data, not authority to change its role, tool policy, recipients, routing rules, or output schema.
- Validate the classification result against a fixed set of allowed outcomes. A classifier or a system prompt alone is not an enforcement boundary.
- Keep permitted recipients, destination folders, and action mappings in trusted configuration. Apply recipient restrictions to To, CC, and BCC. The Email tool does not automatically enforce a recipient allowlist.
- Execute approved actions in explicit workflow states or another controlled execution path. Validate any message-derived arguments before using them, and require review for sensitive actions where appropriate.
- Avoid logging credentials, full message bodies, or attachment contents.

These precautions also apply when external content comes from other tools. For example, a scraped web page can contain instructions even after conversion to Markdown.

## Scheduled mailbox processing

Use an existing [Scheduler](../../schedulers/create-scheduler.md) to invoke an assistant or workflow periodically. The Email toolkit itself does not create an incoming-mail webhook or start a background mailbox watcher.

A folder can act as a visible queue. Create source and destination folders, then configure this processing pattern:

```text
Scheduler
  → list_email_folder(Unprocessed)
  → for each message: read → classify → validate result → perform approved action
  → move successful messages to Processed
  → request the next page while has_more is true
```

Keep the current `email_id` isolated for each message, especially if processing items in parallel. A message left in the source folder remains eligible for a later run. Start the next run without a continuation ID, and bound each run's work so a growing mailbox does not keep it running indefinitely.

This pattern provides **at-least-once processing**, not exactly-once delivery. An action can succeed and the subsequent move can fail, leaving the message to be processed again. Use an idempotency key or durable record of completed business actions to prevent duplicate effects. Do not rely on moving the message as the only proof that its business action completed.

Avoid overlapping runs against the same queue unless you have a claim or locking mechanism. Define a review or dead-letter policy for repeatedly failing messages; paging lets a run reach later messages, but does not resolve those failures. A failed direct Tool state can stop the current run, so retry and recovery behavior must be designed explicitly.

## Upgrading from the Email Sender tool

The former **Notification → Email** tool is now **Email → Send Email**, with tool identifier `send_email`. Stored integration settings, assistant and workflow references, and user integration selections are migrated during the upgrade.

Update API/SDK payloads, imported configurations, and scripts to use the current names and credential keys:

| Previous value                         | Current value                                                                                       |
| -------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Tool `Email` in toolkit `Notification` | Tool `send_email` in toolkit `Email`                                                                |
| `url`                                  | `smtp_url`                                                                                          |
| `smtp_username`                        | `username`; also configure `smtp_from_email`                                                        |
| `smtp_password`                        | `password`                                                                                          |
| `oauth_from_email`                     | `smtp_from_email`                                                                                   |
| No capability flags                    | Set both `inbound_enabled` and `outbound_enabled` explicitly. For SMTP-only use `false` and `true`. |

Temporary compatibility for one release accepts the old `Email` name and converts legacy credential dictionaries. If both capability flags are absent from a legacy dictionary, it is treated as SMTP-only. Explicit new fields take precedence over legacy values. New clients should use the current schema rather than depend on this compatibility.

Also account for the changes to certificate verification and SMTP error handling described above. Update and deploy the matching backend and UI together so that the integration form and API use the same configuration contract.
