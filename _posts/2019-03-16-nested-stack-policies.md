---
layout: post
date: 2019-03-16T00:00:00.000Z
excerpt_separator: <!--more-->
tags:
  - AWS
  - bash
  - cloudformation
title: >-
  Dude, where's my database? Preventing disasters when using CloudFormation
  nested stacks
---

![With great power, comes great responsibility](https://media.giphy.com/media/MCZ39lz83o5lC/giphy.gif)

Managing your infrastructure as code with CloudFormation can be a double edged sword. 

You're able to automate the provisioning of vast amounts of resources from a few YAML files; at the same time a single coding error can result in your production database getting wiped and replaced afresh. 

Fortunately AWS have already considered this and have a CloudFormation feature called [Stack Policies](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html).

<!--more-->

## Stack Policies

Stack Policies are a JSON document you can associate with a stack to define exactly what may be modified, replaced or deleted when applying an update to your CloudFormation stack. For example, take the below stack policy which allows all update actions to all resources, but explicitly prevents replacements or deletions of any database instances:

```json
{
  "Statement" : [
    {
      "Effect" : "Allow",
      "Action" : "Update:*",
      "Principal": "*",
      "Resource" : "*"
    },
    {
      "Effect" : "Deny",
      "Action" : ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource" : "*",
      "Condition": {
      	"StringEquals" : {
          "ResourceType" : ["AWS::RDS::DBInstance"]
        }
      }
    }
  ]
}
```

You can associate these policies during stack creation:
```
aws cloudformation create-stack --stack-name prodstack --stack-policy-body file://stackpolicy.json <...>
```

or to an existing stack:
```
aws cloudformation set-stack-policy --stack-name prodstack --stack-policy-body file://stackpolicy.json
```

Problem solved, right?

![hahaha no](https://media.giphy.com/media/SdYnnxQ30OahG/giphy.gif)
*AWS destroying your unprotected resources in your nested stacks*

## Nested Stacks

Not if you follow [AWS CloudFormation best practice](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html#nested) and use nested stacks.

Just like it is good practice to break out a large application into composable modules which each have a single responsibility, you shouldn't cram your entire infrastructure into a 5000 line long YAML file.

![Nested Stack AWS Diagram](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/images/cfn-console-nested-stacks.png)
*Image from [AWS](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html)*

> As your infrastructure grows, common patterns can emerge in which you declare the same components in multiple templates. You can separate out these common components and create dedicated templates for them. Then use the resource in your template to reference other templates, creating nested stacks.

So you end up with a root stack, which in turn contains other stacks which are these individual, reusable components.

The problem is that when you create that root stack and attach a stack policy to it, **the stack policy doesn't apply down to any of the nested stacks**.

If you follow AWS' best practices, by creating a database component in a nested stack and creating the root stack with a stack policy, you still won't be protected from an accidental deletion of your database.

## The Solution

*After* creating, or *before* updating a stack which contains nested stacks, use the CloudFormation API to retrieve all of the nested stacks and apply your stack policy individually to them.

```bash
function protect_nested_stacks() {
    local parent_stack=$1 stack_policy_file=$2
    local nested_stacks=$(aws cloudformation list-stack-resources \
        --stack-name ${parent_stack} \
        --query "StackResourceSummaries[?ResourceType=='AWS::CloudFormation::Stack'].[PhysicalResourceId]" \
        --output text
    )
    
    for stack in ${nested_stacks}; do
        aws cloudformation set-stack-policy --stack-name ${stack} --stack-policy-body file://${stack_policy_file}
        protect_nested_stacks ${stack} ${stack_policy_file}
    done
}
```

You can call this function like: `protect_nested_stacks ParentStackName StackPolicyFile.json`, of course replacing `ParentStackName` and `StackPolicyFile.json` with your actual values.

We list all of the resources created by the parent stack, and filter out any which are nested stacks (`AWS::CloudFormation::Stack`), retrieving the stack IDs. We then loop over the IDs and apply the stack policy to each nested stack.

To handle the fact that there might be more than 1 level of nested stacks, we then recursively call the `protect_nested_stacks` function in case any of your nested stacks contain other nested stacks.

![I heard you like nested stacks](https://media.giphy.com/media/ZLWnbaMlDjzGg/giphy.gif)
*It's nested stacks all the way down*

## In Conclusion

We're still operating with relatively young technologies. The tools we use to spin up infrastructure from code are very powerful, but there's some nasty gotchas you need to be aware of.

It would be great if AWS could apply the same stack policy on the root stack to the nested stacks by default. If customisation was needed on a stack by stack basis, perhaps a `StackPolicy` parameter could be added to the `AWS::CloudFormation::Stack` resource.

I've reached out to AWS about this and they've confirmed it's something that customers have been requesting, so lets hope we see a native AWS solution for this some time soon!
