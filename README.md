```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSpecificIAMRolesAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::521418994264:role/ADFS-AppDevOps",
                    "arn:aws:iam::521418994264:role/ADFS-InfraDevOps"
                ]
            },
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::insp-life-datamig-unmasked-rls-521418994264-ap-east-1",
                "arn:aws:s3:::insp-life-datamig-unmasked-rls-521418994264-ap-east-1/*"
            ]
        },
        {
            "Sid": "DenyAllOtherPrincipals",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::insp-life-datamig-unmasked-rls-521418994264-ap-east-1",
                "arn:aws:s3:::insp-life-datamig-unmasked-rls-521418994264-ap-east-1/*"
            ],
            "Condition": {
                "StringNotLike": {
                    "aws:PrincipalArn": [
                        "arn:aws:iam::521418994264:role/ADFS-AppDevOps",
                        "arn:aws:iam::521418994264:role/ADFS-InfraDevOps"
                    ]
                }
            }
        }
    ]
}
