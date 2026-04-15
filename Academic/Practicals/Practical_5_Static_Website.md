# Practical 5 — Deploy a Static Website on S3
**Objective:** Host a static website on AWS S3 with public access, custom error pages, and understand S3 architecture.

---

## All Terms You Will Encounter

| Term | Definition |
|------|-----------|
| **S3** | Simple Storage Service — AWS object storage |
| **Bucket** | Container for objects (files) in S3. Globally unique name required |
| **Object** | A file stored in S3 (data + metadata + key) |
| **Key** | The unique identifier (path) of an object: `images/logo.png` |
| **ACL** | Access Control List — legacy per-object/bucket permissions |
| **Bucket Policy** | JSON document defining who can access what in the bucket |
| **Static Website** | HTML/CSS/JS files served directly — no server-side code execution |
| **Index Document** | Default file served when accessing root URL (e.g., `index.html`) |
| **Error Document** | File served for 404 errors (e.g., `error.html`) |
| **Public Access** | Bucket/objects accessible by anyone on the internet |
| **Block Public Access** | AWS account-level setting that prevents accidental public exposure |
| **Website Endpoint** | Special URL format for S3 static hosting (not the same as object URL) |
| **CloudFront** | AWS CDN — sits in front of S3 for HTTPS + global caching |
| **Object URL** | Direct link to a specific object |
| **Versioning** | Keep multiple versions of objects; recover from overwrites/deletions |
| **Storage Class** | Tiering system: Standard → IA → Glacier (lower cost = slower access) |
| **Durability** | How safe data is (S3 = 99.999999999% — 11 nines) |
| **Availability** | How often you can access data (S3 Standard = 99.99%) |

---

## S3 Architecture for Static Website Hosting

```
┌──────────────────────────────────────────────────────────────────────┐
│              S3 STATIC WEBSITE ARCHITECTURE                          │
│                                                                      │
│  WITHOUT CLOUDFRONT (HTTP only, one region):                        │
│                                                                      │
│  User Browser                                                        │
│       │                                                              │
│       │ HTTP GET http://my-bucket.s3-website-ap-south-1.amazonaws.com│
│       ▼                                                              │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  S3 Bucket: my-website-bucket                                │   │
│  │  Region: ap-south-1 (Mumbai)                                 │   │
│  │                                                              │   │
│  │  Static Website Hosting: ENABLED                             │   │
│  │  Index document: index.html                                  │   │
│  │  Error document: error.html                                  │   │
│  │                                                              │   │
│  │  Objects:                                                    │   │
│  │  ├── index.html        (served on /)                        │   │
│  │  ├── error.html        (served on 404)                      │   │
│  │  ├── style.css                                               │   │
│  │  └── images/banner.jpg                                       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  WITH CLOUDFRONT (HTTPS + global CDN + custom domain):              │
│                                                                      │
│  User Browser → CloudFront Edge (Mumbai/London/NY) → S3 Origin     │
│                  (Cached here, fast for all users)                   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step: Host Static Website on S3

### Step 1: Create the S3 Bucket

1. AWS Console → S3 → **Create bucket**
2. Bucket name: `my-static-website-[yourname]` (must be globally unique)
3. Region: `ap-south-1` (or your preferred)
4. **Block Public Access:** UNCHECK "Block all public access"
5. Acknowledge the warning checkbox
6. Leave all other settings default
7. **Create bucket**

**Why globally unique names?** S3 bucket names form part of URLs. All AWS accounts share the same namespace. If `my-bucket` exists in any account anywhere, no one else can use that name.

---

### Step 2: Create Website Files

Create these files on your local machine:

**index.html:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My Cloud Website</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #f0f0f0; }
        h1 { color: #232f3e; }
        p  { color: #555; }
        .badge { background: #ff9900; color: white; padding: 8px 16px; border-radius: 4px; }
    </style>
</head>
<body>
    <h1>Hello from AWS S3!</h1>
    <p>This static website is hosted on Amazon S3.</p>
    <p><span class="badge">IaaS → Storage → S3</span></p>
    <p>No servers. No maintenance. Just HTML served from object storage.</p>
</body>
</html>
```

**error.html:**
```html
<!DOCTYPE html>
<html>
<head><title>404 - Page Not Found</title></head>
<body>
    <h1>404 - Page Not Found</h1>
    <p>The page you requested does not exist.</p>
    <a href="/">Go Home</a>
</body>
</html>
```

---

### Step 3: Upload Files to S3

1. Click on your bucket → **Upload**
2. Click "Add files" → Select both `index.html` and `error.html`
3. Leave all settings default
4. **Upload**

---

### Step 4: Enable Static Website Hosting

1. Bucket → **Properties** tab
2. Scroll to "Static website hosting" → **Edit**
3. Enable: **Enable**
4. Hosting type: **Host a static website**
5. Index document: `index.html`
6. Error document: `error.html`
7. Save changes
8. Note the **Bucket website endpoint** URL at the bottom of Properties tab

---

### Step 5: Add Bucket Policy for Public Read

1. Bucket → **Permissions** tab → **Bucket Policy** → **Edit**
2. Paste this policy (replace `BUCKET-NAME` with your actual bucket name):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::BUCKET-NAME/*"
        }
    ]
}
```

**Policy breakdown:**
- `"Effect": "Allow"` — this grants permission
- `"Principal": "*"` — to ANYONE (public internet)
- `"Action": "s3:GetObject"` — to READ objects only (not upload, delete)
- `"Resource": "arn:aws:s3:::BUCKET-NAME/*"` — on all objects in this bucket

3. Save

---

### Step 6: Test the Website

1. Go to Properties → Static website hosting → Copy the endpoint URL
2. Looks like: `http://my-static-website-yourname.s3-website-ap-south-1.amazonaws.com`
3. Open in browser → You should see your index.html
4. Try: `http://[endpoint]/doesnotexist.html` → Should show error.html

---

## S3 Object Storage vs Block Storage — Key Distinction

```
┌──────────────────────────────────────────────────────────────────┐
│              OBJECT vs BLOCK STORAGE                             │
│                                                                  │
│  S3 OBJECT STORAGE:             EBS BLOCK STORAGE:             │
│                                                                  │
│  ┌────────────────────┐         ┌────────────────────┐          │
│  │  Object            │         │  Block Device      │          │
│  │  Key: photo.jpg    │         │  Like a hard disk  │          │
│  │  Data: [bytes]     │         │  Mounted on EC2    │          │
│  │  Metadata: {size,  │         │  Supports: read,   │          │
│  │  content-type...}  │         │  write, seek       │          │
│  └────────────────────┘         └────────────────────┘          │
│                                                                  │
│  Access: HTTP URL               Access: File system path        │
│  Operations: PUT, GET, DELETE   Operations: Read/Write bytes    │
│  Use: Files, images, backups    Use: OS, database, app storage  │
│  Size: Unlimited                Size: Up to 64TB per volume     │
│  Concurrent: Many readers       Concurrent: One EC2 writer      │
└──────────────────────────────────────────────────────────────────┘
```

---

## Viva Questions — Practical 5

1. **What is the difference between the S3 object URL and the website endpoint URL?**  
   A: Object URL (`https://bucket.s3.amazonaws.com/file.html`) — direct HTTPS link to object, requires bucket policy for public access. Website endpoint (`http://bucket.s3-website-region.amazonaws.com`) — activates static hosting, serves index.html on root, handles error.html for 404s. Note: website endpoint is HTTP only, not HTTPS.

2. **Why can't you run PHP/Python/Node.js code on S3 static hosting?**  
   A: S3 serves files — it doesn't execute code. It's object storage, not a web server. For server-side code you need EC2, Lambda, or Elastic Beanstalk.

3. **What is the "Block Public Access" setting?**  
   A: Account-level and bucket-level safeguard preventing accidental public exposure. On by default. Must explicitly disable it before any public bucket policy takes effect. Designed to prevent the common mistake of accidentally making sensitive data public.

4. **What is the ARN in the bucket policy?**  
   A: Amazon Resource Name — unique identifier for any AWS resource. Format: `arn:partition:service:region:account-id:resource`. For S3: `arn:aws:s3:::bucket-name/*` (no region/account because S3 is global).

5. **Why is S3 durability 11 nines (99.999999999%)?**  
   A: AWS stores each object across at least 3 AZs (actually more). Each copy on different physical hardware. To lose data, multiple AZ-level disasters would need to happen simultaneously — statistically near-impossible.

6. **What is S3 versioning? When would you enable it?**  
   A: Keeps all versions of objects. If you upload a new version of a file or delete one, previous version is preserved. Enable for: production data that must not be accidentally overwritten/deleted, compliance requirements, backup strategy.

7. **What is the difference between S3 Standard and S3 Glacier?**  
   A: Standard = frequent access, millisecond retrieval, highest cost. Glacier = archival, minutes-to-hours retrieval, very cheap. Use Glacier for: backups you rarely need, compliance archives. Lifecycle policies can auto-move old objects from Standard to Glacier.

8. **Can S3 host websites with custom domains (e.g., www.mysite.com)?**  
   A: Yes, with CloudFront + Route 53. S3 website endpoint is HTTP only. CloudFront CDN sits in front: provides HTTPS, custom domain, global edge caching, and security (WAF integration).

---
