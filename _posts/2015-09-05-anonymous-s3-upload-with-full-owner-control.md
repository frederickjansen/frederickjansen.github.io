---
layout: post
title: Anonymous S3 file upload with full bucket owner control
excerpt: With the right ACL policy and settings you can allow anonymous users to upload files to S3, without giving them any other permissions (such as delete), and handing over control to the bucket owner.
---

I recently had to create a file upload service for anonymous users, where they had no permission to view their own files nor delete them. The S3 bucket owner on the other hand should have full rights. It took me a while to figure out how. [This tutorial](https://gist.github.com/jareware/d7a817a08e9eae51a7ea) for example accomplishes the first and second part, but it requires a special token for the bucket owner to do anything. This is in part because the bucket owner is not the owner of the file, but also because a Deny rule would target them without the token. This means that you need a special client (or curl) to send this token, and the owner is basically locked out of the [AWS S3 portal](https://console.aws.amazon.com/s3/home).  

A better way of doing this then is as follows:

## Create a bucket

The first step is to create a bucket with S3. Make note of the bucket name, you'll need it later on. In my case you'll see my bucket name listed as *abc123* below.

## Add a bucket policy

You can find more information on Access Control Lists (ACL) in [Amazon's AWS documentation](https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html). The first part is allowing anonymous users to upload to your bucket.

{% highlight javascript %}
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "allow-anon-put",
			"Effect": "Allow",
			"Principal": {
				"AWS": "*"
			},
			"Action": "s3:PutObject",
			"Resource": "arn:aws:s3:::abc123/*"
		}
	]
}
{% endhighlight %}

The key part here is *AWS: \**, indicating all users, even the anonymous ones. This works, but you'll get into trouble later down the line. Even as a bucket owner, by default you won't have unrestricted access to the files other people upload. There's bucket ownership and then there's file ownership. You want to get both, so we adjust the policy with a condition.

{% highlight javascript %}
"Condition": {
	"StringEquals": {
		"s3:x-amz-acl": "bucket-owner-full-control"
	}
}
{% endhighlight %}

Now the anonymous user is forced to add a header that tells Amazon the bucket owner should have full control as well. Next we want to make sure that no users apart from the bucket owner have any rights.

{% highlight javascript %}
{
	"Sid": "deny-other-actions",
	"Effect": "Deny",
	"NotPrincipal": {
		"AWS": "USER_ID"
	},
	"NotAction": [
		"s3:PutObject",
		"s3:PutObjectAcl"
	],
	"Resource": "arn:aws:s3:::abc123/*"
}
{% endhighlight %}

When you use NotPrincipal in combination with Deny the policy statement is denied to all users except those [listed in the principal](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html#NotPrincipal). *USER_ID* should be replaced with your account's user ID. This you can find in your [account settings](https://console.aws.amazon.com/billing/home?#/account).  
With NotAction all actions apart from the ones listed are denied. In this case you need PutObjectAcl because the uploader needs to be able to grant access to the bucket owner. Without this the upload would fail. Bringing it all together, all users but the bucket owner (whose ID you supply) are denied entry for all actions apart from the ones listed.

Here is the full policy:
{% highlight javascript %}
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "allow-anon-put",
			"Effect": "Allow",
			"Principal": {
				"AWS": "*"
			},
			"Action": "s3:PutObject",
			"Resource": "arn:aws:s3:::abc123/*",
			"Condition": {
				"StringEquals": {
					"s3:x-amz-acl": "bucket-owner-full-control"
				}
			}
		},
		{
			"Sid": "deny-other-actions",
			"Effect": "Deny",
			"NotPrincipal": {
				"AWS": "USER_ID"
			},
			"NotAction": [
				"s3:PutObject",
				"s3:PutObjectAcl"
			],
			"Resource": "arn:aws:s3:::abc123/*"
		}
	]
}
{% endhighlight %}

## Additional settings

While anonymous users are not capable of deleting files outright, Amazon does not offer a policy that prohibits overwriting files. Nothing stops anyone from overwriting a file with one that has the same file name and size 0. This is where [versioning](https://docs.aws.amazon.com/AmazonS3/latest/UG/enable-bucket-versioning.html) comes into play. Files can no longer be outright overwritten, but instead a copy of the old file will be made. With [Lifecycle Management](https://docs.aws.amazon.com/AmazonS3/latest/dev/object-lifecycle-mgmt.html) you can limit the amount of files that can be stored this way.