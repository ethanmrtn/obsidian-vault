This is my vault, I've just started using it.

As I'm keeping this mainly as a log of all my learnings, I figured it made sense as the first step it be to work out how to host my own obsidian vault. I'm in the process of moving all my hosting to AWS so makes sense to try self host using that. A quick google showed a few options for self hosting on a local machine that seem pretty widely used. They all use this community plugin:
https://github.com/vrtmrz/obsidian-livesync
Live sync works with S3 out of the box, it’s pretty easy, you just setup an s3 bucket, I do this kinda stuff via terraform
``` terraform
resource "aws_s3_bucket" "livesync" {
  bucket = "obsidian-livesync-ethanmrtin"

  tags = {
    Name = "obsidian-livesync"
  }
}

resource "aws_s3_bucket_cors_configuration" "livesync" {
  bucket = aws_s3_bucket.livesync.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST", "DELETE", "HEAD"]
    allowed_origins = ["app://obsidian.md", "capacitor://localhost", "http://localhost"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3600
  }
}

resource "aws_s3_bucket_versioning" "livesync" {
  bucket = aws_s3_bucket.livesync.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

Obviously it’s not supposed to be open to everyone on the internet, live sync just wants an access key and a secret, which we can give it by setting up an iam role
``` terraform
resource "aws_iam_user" "livesync" {
  name = "obsidian-livesync-s3-user"

  tags = {
    Name = "obsidian-livesync-s3-user"
  }
}

resource "aws_iam_access_key" "livesync" {
  user = aws_iam_user.livesync.name
}

resource "aws_iam_user_policy" "livesync" {
  name = "obsidian-livesync-s3-policy"
  user = aws_iam_user.livesync.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.livesync.arn,
          "${aws_s3_bucket.livesync.arn}/*"
        ]
      }
    ]
  })
}

output "access_key_id" {
  value = aws_iam_access_key.livesync.id
}

output "secret_access_key" {
  value     = aws_iam_access_key.livesync.secret
  sensitive = true
}
```
Then all you gotta do is follow the settings in live sync to setup the s3 remote vault. Pretty cool plugin, anyone that ends up reading this (which I’m not too sure why you are cause this is super lazily written while I watch Mr Deeds next to my sick girlfriend at 10pm on a Saturday night) should give it a star on gh.