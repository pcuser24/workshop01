---
title : "IAM user"
date : "`r Sys.Date()`"
weight : 52
chapter : false
--- 

## IAM User
Remove the IAM user created for this workshop:
1. Go to the [IAM console](https://console.aws.amazon.com/iam/home).
2. In the navigation pane, choose **Users**.
3. Select the user you created for this workshop.
4. Choose **Delete user**.
5. In the confirmation dialog box, choose **Yes, delete**.
6. Delete the access keys associated with the user.
7. Choose **Delete**.
8. In the confirmation dialog box, choose **Yes, delete**.
9. Delete the user's policies.
10. Choose **Detach policy**.
11. Select the policies attached to the user.
12. Choose **Detach policy**.
13. Choose **Delete policy**.
14. In the confirmation dialog box, choose **Delete**.
15. Repeat the above steps for all the policies attached to the user.
