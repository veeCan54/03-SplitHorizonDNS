# Split Horizon or Split View DNS implementation
Using the Split Horizon facility of DNS, it is possible to return different sets of DNS information, usually selected by the source address of the DNS request. This facility can provide a mechanism for security and privacy management by logical or physical separation of DNS information for network-internal access (within an administrative domain, e.g., company) and access from an unsecure, public network (e.g. the Internet).  
 
Split view DNS can be implemented with hardware based separation or software solutions. Using Route 53 Split Horizon or Split view architecture, we can have internal applications in a VPC resolve to internal only DNS records while external users would be redirected to the external facing web site. In this simple implementation we are doing it by using 2 hosted zones of the same name, one public and one private. The public hosted zone will host the record for the external site and private hosted zone will host the record for the internal website. External users will be taken to a corporate web page served by Apache web server running on EC2. Internal users will be taken to a static employee website hosted on S3. 
 
# Steps : 
1. Create Custom VPC.  [Details](#Step1)
2. Test to make sure we can view the Corporate web page via a browser using the public IP address of the EC2 instance. [Details](#Step2)
3. Set up public hosted zone in Route 53. [Details](#Step3)
4. Test the url using a browser and using dig command from the public internet. [Details](#Step4)
5. Test results from inside the VPC. [Details](#Step5)
6. We need an S3 static website to serve as the internal employee website. Enable static website hostng & make sure it is working. [Details](#Step6)
7. Create a private hosted zone with the same name as public hosted zone. [Details](#Step7). 
8. Create a CNAME record in the private hosted zone pointing to the static website url. [Details](#Step8)
9. Test the difference in behavior when accessed from the internet vs from inside the VPC (intranet). [Details](#Step9)
10. Clean up the resources by deleting the stack. Empty the contents of the S3 bucket and delete the bucket. If we want to keep the hosted zones in Route 53 it should cost us about $0.50 per month.
11. Summary & What I learned. [Details](#summary)

# Implementation steps:
# Step1:  
Create the VPC using the Cloudformation Template [here](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/files/01-SingleCustomVPCWithPublicSubnet.yml).  
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step1-CustomVPC.png) 
# Step2: 
Test to make sure the Corporate website is accessible using the public IP address of the EC2 instance. 
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step2.png)
# Step3:  
Create a public hosted zone in Route 53. I already had a domain name purchased via Route 53 so I have a Route 53 public zone in my AWS account.  
In the public zone create a record with simple routing pointing to the public IP address of EC2 instance.
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step3.png)
# Step4: 
Test to make sure it resolves correctly via the browser.
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step4-1.png)
Another way to test it is using the terminal. Use the dig or curl command. 
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step4.png)
# Step5: 
Connect to the EC2 instance using Instance Connect and do the same. We get the same results.
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step5.png)
# Step6: 
Create an S3 bucket with the same name as the public hosted zone. Make sure it is set to public access. 
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step5-S3BucketSettings.png)
Enable S3 static site hosting under properties.
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step5-enableStaticWebsite.png)
Apply the bucket policy.  
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicRead",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::yourbucketname/*"
        }
    ]
}
```
Download the [2 files here](https://github.com/veeCan54/03-SplitHorizonDNS/tree/main/s3static_internal) and upload these files to the S3 bucket.
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step5-BucketUpload.png)  
Test the S3 static website via the browser to make sure we get the employee section.
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step8-StaticWebsite.png)
# Step7:   
Create a private hosted zone with the same name as public hosted zone and associate it with MyCustomVPC. 
The one click deployment template sets both DNS flags in the VPC to true so we already have that in place in our custom VPC.  
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step1-EnableDNSSettings.png) 
The private hosted zone can be accessed only from the VPC it is associated with so make sure this is set correctly. 
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step8-privateHostedZone.png)
# Step8:  
In this zone create a CNAME record pointing www to the static website URL for the employee website. Set the TTL to be 60 sec.
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step7-CNAMERecord.png)
# Step9:
Connect to the EC2 instance and access the endpoint. This time it resolves to the internal website. 
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step8-EC2AfterPrivateZone.png)
Both commands produce the same results:
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step8-Ec2AfterPrivatezone2.png)
Go back to the external browser and test it out, nothing has changed here. 
![Alt text](https://github.com/veeCan54/03-SplitHorizonDNS/blob/main/images/Step9-Corporate.png)
# Step10: 
Delete the stack. Empty the S3 bucket and delete it.

## Summary<a name="summary"></a>

**What did I learn?**  
1. Implemented a use case for Split View DNS.
2. This architecture could also be used when we want to redirect a canary release internally first before rolling it out to the users.  
 
**No Mistakes this time - bonus points instead**  
1. I made a small next step in automating my infrastructure. Now it is possible to reuse the corporate website from my git repository in any future projects.  
2. ***Edit: After I finished this is when I realized that in order to have HTTPS enabled on a static website hosted on S3, we need to be using CloudFront**. 
 
**TODO?**  
1. More hands on projects. Networking, here I come.
2. Host an HTTPS website on S3 integrating with CloudFront and ACM.

