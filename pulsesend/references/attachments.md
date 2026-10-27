# Attachments

Three ways to attach a file to `POST /v1/emails`:

- **Inline base64** (one-off files, such as an invoice you just rendered):
  `"attachments": [{ "filename": "receipt.pdf", "content": "<base64>" }]`
- **Files API** (a file sent again and again, such as your terms): upload once with
  `POST /v1/files` (multipart field `file`, or JSON with `filename` and base64 `content`), then
  `"attachments": [{ "file_id": "file_…" }]`. Uploading the same content again returns the existing file.
- **Inline image** (shown in the HTML): give the file a `content_id` and reference it with
  `<img src="cid:logo">`. In multipart send it as `inline[logo]=@logo.png`; in JSON, add `"content_id": "logo"` to the item.

Rules:

- `filename` needs an extension. The content must match it for PDF, PNG, JPEG, GIF, WebP, ZIP, DOCX,
  XLSX, PPTX, ODT and ODS, otherwise `attachment_content_mismatch`.
- Executables and scripts (`.exe`, `.bat`, `.js`, `.vbs`, `.ps1`, `.msi`, `.mdb`…) are refused with
  `attachment_type_not_allowed`. Zip them or send a link.
- Max 20 attachments per email, 25 MB for the whole message (`message_too_large`). Each `content_id` is unique.
- Beyond 1 MB per email, each recipient costs 1 credit per MB started.
- `store: false` deletes the bytes once the email is handed to the provider. Otherwise they are kept
  and purged with the email content after 30 days.
- `GET /v1/emails/{id}/attachments` returns a `download_url` valid for 5 minutes.

Never retry an attachment error without changing the request: it fails the same way every time.
Full guide: https://docs.pulse-send.jiordiviera.me/guides/attachments
