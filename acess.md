> An access point is an application-specific entry point into an EFS file system that makes it easier to manage application access to shared datasets.

Recently, I got a call from one of my customers, saying they were struggling to set up a S3 access point. They wanted to share a large data set across several company accounts, but could not get the permissions to work correctly. After some back and forth, I managed to weed out all the issues. However, we ended up spending much more time than any of us would like to admit.
To avoid this happening in the future, I decided to write a follow-up, which summarizes main points and caveats regarding the S3 access points permissions.

# The requirement
We have a S3 bucket, which contains several large data sets that are organized in different namespaces. Essentially, each data set has its own key prefix, i.e., its own folder.
We have two main requirements: i) Provide access to one of the data sets (objects in one of the bucket’s folders) and ii) Enable listing all objects (only) from that same folder.
Setting up multi-account access for S3 Access Point

# Scenario
Lets assume we have a bucket (test-bucket) in AccountA and we have created an access point (test-ap) in the same account. In another account AccountB, we have an IAM user, whom we will call UserInAccountB.
Note: usually we would have an IAM role instead of an user, but for the sake of simplicity, in this article we will consider only the case of one IAM user.
In our test-bucket, let’s also create two folders public and private and add a testObject to both of them. We will use this to test our setup in the end.
Our goal is to enable the user UserInAccountB to access the data in the public folder of our test-bucket, through the test-ap. Let’s now see how to do that.
Access policies
To achieve our goal, we need to create three policies. First, we need to create an IAM policy in the AccountB. It should look similar to the policy shown below and the IAM policy should be attached to our UserInAccountB.

The IAM policy for a user in AccountB.
This policy grants full S3 access¹ to our test-ap access point in AccountA. Note that at this stage we also need to provide access to the underlying bucket (we will block direct bucket in a few moments). We can further limit the access controls by specifying only some actions we want to allow, such as s3:GetObject or s3:ListBucket. With this we are done setting up things in the AccountB.
Further, we need two policies in the AccountA: One S3 bucket policy and one S3 Access Point policy. Our S3 bucket policy needs to be attached to our S3 bucket test-bucket and it should look something like this:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::test-bucket",
                "arn:aws:s3:::test-bucket/*",
                "arn:aws:s3:us-east-1:AccountA:accesspoint/test-ap",
                "arn:aws:s3:us-east-1:AccountA:accesspoint/test-ap/*"
            ]
        }
    ]
}
```
# S3 bucket policy for the bucket in AccountA.
This bucket allows any action, as long as it is coming form an access point created for that bucket. However, direct access to the bucket is not allowed, thus it will be denied. Effectively, with this pattern we delegate bucket’s permissions management to the access point. Otherwise, we would have to manage a both bucket’s and access point’s permissions in parallel. This is because S3 Access Points are simply named network endpoints, which are attached to S3 buckets. Attaching an access point to a bucket does not change anything about the underlying bucket. All existing operations against the bucket continue to work as before. Restrictions that are included in an access point policy apply only to requests made through that access point. That is why we would need to allow the desired operations both on the bucket as well as on the access point level, i.e., maintain both polices in parallel.
Finally, we need to create the following S3 access point policy and attach it to our access point.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "*",
            "Resource": [
                "arn:aws:s3:::test-bucket",
                "arn:aws:s3:::test-bucket/*"
            ],
            "Condition": {
                "StringEquals": {
                    "s3:DataAccessPointAccount": "AccountA"
                }
            }
        }
    ]
}
```
S3 Access Point policy for the access point in AccountA.
This policy is at the core of our permissions management. It has two statements: AllowListFolderToIamUser, which enables the folder listing and AllowReadAccessToIamUser, which provides access to the objects. Firstly, we notice that both statements reference our UserInAccountB as the principal. This is the second piece of the puzzle, how we enable cross-account access to an S3 access point. Secondly, to satisfy our requirement of only giving access to the public folder, we add a condition “s3:prefix”: “public/* to check if the request specifies an allowed path. Finally, in AllowReadAccessToIamUser statement, we need to make sure that we grant read access only to the objects in the same public folder. Note that S3 access point syntax requires us to add objects/ prefix in our resource specification.
```
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "AllowListFolderToIamUser",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam:AccountB:user/UserInAccountB"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:us-east-1:AccountA:accesspoint/test-ap",
            "Condition": {
                "StringLike": {
                    "s3:prefix": "public/*"
                }
            }
        },
        {
            "Sid": "AllowReadAccessToIamUser",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::user/UserInAccountB"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:us-east-1:AccountA:accesspoint/test-ap/object/public/*"
        }
    ]
}
```

# Testing the setup
Firstly, lets check if the folder listing is working as expected. To test that we run can the following command: aws s3 ls s3://arn:aws:s3:us-east-1:AccountA:accesspoint/test-ap/public/ --profile AccountB-Profile we will see our testObject listed. If we run the same command against our private folder (aws s3 ls s3://arn:aws:s3:us-east-1:AccountA:accesspoint/test-ap/private/ --profile AccountB-Profile) we will get an Access Denied. OK, that is great!
What about accessing the objects? Well, to download our testObject locally, we can run: aws s3 cp s3://arn:aws:s3:us-east-1:AccountA:accesspoint/test-ap/public/testObject . --profile AccountB-Profile. That should download the object to our working directory. The same command should fail with Access Denied if run against the private folder.
Finally, with our UserInAccountB we can try running any s3 command directly against our bucket and we will see that all of them will be denied, since they are not coming through our access point.
