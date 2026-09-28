# Media and files

**Owns:** upload permission, image processing, and the split between public images and private KYC files.  
**Service:** `media-service` for product images. Seller KYC bytes are in a private bucket that only `seller-service` can read.  
**Does not own:** the product row that points at an image id (catalogue).

The browser uploads straight to S3 with a presigned URL. The API does not receive the file body.

---

## Product images

1. Seller asks for an upload. The API checks the seller is approved, the content type is JPEG, PNG, or WebP, and the declared size is at most 8 MB.
2. `media-service` returns a presigned PUT of 10 minutes for `buyymart-prod-images`.
3. The browser uploads to S3.
4. A worker checks the object, strips metadata, writes a WebP rendition at a fixed width for the product page and a smaller one for the listing card, and publishes `MediaReady`.
5. Catalogue attaches the image id to the variant. A product cannot go live with zero ready images.

Public objects are readable only through CloudFront. The bucket itself is not a public website.

Rejected uploads (wrong type, too large, or a file that is not an image) never become `MediaReady`. The seller sees “Upload failed” and can try again.

---

## KYC files

KYC uses `buyymart-prod-kyc`. The bucket blocks public access and uses its own KMS key. Only the seller-service task role can read it.

The seller uploads PAN, GST certificate, and a cancelled cheque or bank proof the same way: presigned PUT, then a private object key stored on the KYC row. Seller Centre and Admin never receive a permanent URL. A reviewer gets a presigned GET of 60 seconds.

KYC objects are not processed into WebP and are not served from the image CDN.

---

## Invoices

Invoice PDFs are written to `buyymart-prod-invoices` by the order and settlement flow, with object lock so a tax invoice cannot be quietly replaced. Retention follows the company policy in the startup documents (the architecture target is 8 years). Media service does not generate invoices.

---

## Rules

- Filenames from the browser are not used as object keys. Keys are generated ids.
- Logs record the object key and the seller id, not the document contents.
- A deleted product hides the catalogue row. The image can be deleted after no live or draft product references it. Images referenced by an old order snapshot are kept.
