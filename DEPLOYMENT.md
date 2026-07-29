# Cortexa Site — AWS Deployment

Static one-page brand site, deployed to S3 + CloudFront, served on `https://thecortexa.com`.

## Live URLs
- **Primary:** https://thecortexa.com
- **WWW:** https://www.thecortexa.com
- **CloudFront domain:** https://d2duym3x769aj8.cloudfront.net

## AWS Resources

| Resource | ID / Value |
|---|---|
| AWS Account | 295470186437 |
| S3 Bucket | `cortexa-site-295470186437` (region: us-east-1, private, no public access) |
| CloudFront Distribution ID | `E3DH9XXDF3ABWC` |
| CloudFront Domain | `d2duym3x769aj8.cloudfront.net` |
| Origin Access Control | `E3S8E2CDMJKMMY` (name: `cortexa-oac`) |
| ACM Certificate ARN | `arn:aws:acm:us-east-1:295470186437:certificate/d12e1152-1ec7-4706-9567-48ff02ea7858` (covers `thecortexa.com` + `www.thecortexa.com`) |
| Route53 Hosted Zone | `Z06502662CPK8H3L5L77N` (thecortexa.com) |

## Architecture

```
Browser → thecortexa.com (Route53 A/AAAA alias)
        → CloudFront (E3DH9XXDF3ABWC, HTTPS via ACM cert)
        → S3 bucket (cortexa-site-295470186437, private, OAC-only access)
```

- The S3 bucket has all public access blocked. Only this specific CloudFront
  distribution can read objects, enforced via a bucket policy scoped to its
  distribution ARN.
- TLS is terminated at CloudFront using the ACM certificate (SNI, TLS 1.2+).
- HTTP requests are redirected to HTTPS at the CloudFront layer.
- Route53 has `A` and `AAAA` alias records for both `thecortexa.com` and
  `www.thecortexa.com` pointing at the CloudFront distribution (alias records
  are free and resolve directly to CloudFront's edge IPs — no extra DNS hop).

## What's deployed

Only the files the page actually references were uploaded (not the full
`branding/` folder of draft assets):
- `index.html`
- `branding/brand-wall3.png` (hero background)
- `branding/cortexa-new-profile.png` (Open Graph / Twitter share image)
- `icons/favicon-16x16.png`, `icons/favicon-32x32.png`, `icons/favicon-512.png`, `icons/apple-touch-icon.png`
- `robots.txt`, `sitemap.xml`

## SEO / analytics

- Title, meta description, and meta keywords target "short film," "AI film,"
  "film production" alongside the Cortexa brand name.
- Open Graph + Twitter Card tags use `cortexa-new-profile.png` as the share
  image, so links posted to social platforms show a proper preview card.
- JSON-LD `Organization` schema in the `<head>` links out to the Instagram,
  YouTube, and Facebook profiles for entity/knowledge-graph association.
- `robots.txt` allows all crawlers and points to `sitemap.xml`.
- Google Analytics (gtag.js) is wired up under measurement ID `G-E60LZQXRPR`.

## Redeploying after edits

From the `cortexa-studio` project folder:

```bash
# Sync changed files
aws s3 cp index.html s3://cortexa-site-295470186437/index.html --content-type "text/html"
# (repeat for any other file you change, e.g. a new background image)

# Invalidate CloudFront's cache so the new version serves immediately
aws cloudfront create-invalidation --distribution-id E3DH9XXDF3ABWC --paths "/*"
```

Without the invalidation step, CloudFront may keep serving the old cached
version at edge locations for a while.

## Making DNS/cert changes

- The hosted zone `thecortexa.com` was already present in this AWS account
  before this deployment — no registrar changes were needed, and no existing
  records (there were none besides NS/SOA) were touched or removed.
- If you ever move the domain to a different registrar or DNS provider, the
  four Route53 records to recreate there are `A`/`AAAA` for `thecortexa.com`
  and `www.thecortexa.com`, each pointed (via CNAME if outside Route53) at
  `d2duym3x769aj8.cloudfront.net`.

## Cost

Expect low single-digit dollars per month at most for a small site like this:
- S3: storage + GET requests (fractions of a cent for ~1.7 MB of assets)
- CloudFront: data transfer out (Price Class 100 = US/Canada/Europe edge
  locations only, chosen to keep cost down — switch to `PriceClass_All` in
  the distribution config if you need faster edge delivery in Asia/South
  America/Australia)
- ACM certificate: free
- Route53: hosted zone ($0.50/mo, likely already billed if the zone predates
  this deployment) + negligible query costs

## Rollback / teardown

If you ever need to remove this deployment:

```bash
aws cloudfront get-distribution-config --id E3DH9XXDF3ABWC   # get current ETag
aws cloudfront update-distribution --id E3DH9XXDF3ABWC --distribution-config file://disabled-config.json --if-match <ETag>  # set Enabled: false first
# wait for it to redeploy disabled, then:
aws cloudfront delete-distribution --id E3DH9XXDF3ABWC --if-match <ETag>
aws s3 rm s3://cortexa-site-295470186437 --recursive
aws s3api delete-bucket --bucket cortexa-site-295470186437
aws acm delete-certificate --certificate-arn arn:aws:acm:us-east-1:295470186437:certificate/d12e1152-1ec7-4706-9567-48ff02ea7858 --region us-east-1
# then remove the A/AAAA alias records for thecortexa.com and www.thecortexa.com from Route53
```

CloudFront distributions must be disabled before they can be deleted, and
disabling can take several minutes to propagate.
