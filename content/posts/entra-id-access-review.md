+++
title = 'Entra Id Access Review'
date = 2024-07-06T12:22:31+02:00
draft = false
tags = ['EntraId', 'Identity Governance', 'AccessReview', 'Azure']
+++

Been working on Identity Governance, access review, and I'll admit it can get confusing real fast!

If you're still using excel spreadsheet to keep track who should have access or be a member of a security group, then access review is worth looking into!

You'll need a P2 license to play with Identity governance. I still have access to my dev tenant, better put it to use as long as I can... :wink:

## EntraId Identity Governance

Microsoft does a better job at explaining this:

>[Identity governance](https://learn.microsoft.com/en-us/graph/api/resources/identitygovernance-overview?view=graph-rest-1.0) allows you to balance your organization's need for security and employee productivity with the right processes and visibility. It provides you with capabilities to ensure that the right principals have the right access to the right resources and at the right time.

The part I'm most excited about are the programmatical capabilities, using Microsoft Graph APIs :wink:

PowerShell module _Microsoft.Graph.Identity.Governance_ has all the necessary cmdlets. Just be warned the names can be confusing at times, it is what it is.

## Access Review

As someone who's been in IT well over two decades "*ahem*", I'm all too familiar with maintaining/migrating security group members and reporting who has access or not. Ah the stories I could tell... but I digress...

### Purpose of Access Reviews

Access management & security compliance is the main purpose of access review.

- Security Compliance: Ensure compliance with security policies and regulations by regularly reviewing user access.
- Access Management: Maintain control over who has access to what, minimizing unnecessary or excessive permissions.

Easier said than done. While looking into access review, I learned the hard hard way that there are different types of access reviews, which in hindsight should have been obvious...

### Types of Access Reviews

These are the types we can use:

- Group Membership Reviews: Evaluate the members of security or Microsoft 365 groups.
- AccessPackage Reviews: Review assignments to accessPackages. AccessPackages are a great way to delegate the granting/reviewing process to the owners/delegates
- Privileged roleassignments Reviews. Review assignments to privileged roles. This is great for PIM access reviews

I started looking first at group membership review. Looking at the [documentation](https://learn.microsoft.com/en-us/graph/api/accessreviewset-list-definitions?view=graph-rest-1.0&tabs=powershell) it seems schedule definitions is the place to start. Also, I should have read a bit further... I mean why read if you can learn things the hard way right? :grin:

Running the cmdlet _Get-MgIdentityGovernanceAccessReviewDefinition_ gave me more than just the security groups scheduled definitions I saw in my access review blade. The access review interacts with the above mentioned types :smile:

My first attempt was to filter on the displayName or id.

```pwsh
function Get-AccessReviewDefinition {
    [CmdletBinding(DefaultParameterSetName = 'DisplayName')]
    param (
        [Parameter(ParameterSetName = 'DisplayName')]
        [string]$DisplayName,

        [Parameter(ParameterSetName = 'Id')]
        [guid]$Id
    )

    if ($PSCmdlet.ParameterSetName -eq 'DisplayName') {
        $result = Get-MgIdentityGovernanceAccessReviewDefinition -Filter "DisplayName eq '$DisplayName'"
    }

    if ($PSCmdlet.ParameterSetName -eq 'Id') {
        $result = $result = Get-MgIdentityGovernanceAccessReviewDefinition -AccessReviewScheduleDefinitionId $Id
    }

    $result
}
```

The drawback here is that I need to know either the displayName or id, also this doesn't tell me which type I'm interacting with.

That's where the [$filter](Get-MgIdentityGovernanceAccessReviewDefinition) query parameter comes in. Rule of thumb, $filter early whenever possible. This [example](https://learn.microsoft.com/en-us/graph/api/accessreviewset-list-definitions?view=graph-rest-1.0&tabs=powershell#example-2-retrieve-all-access-review-definitions-scoped-to-all-microsoft-365-groups-in-a-tenant) shows you how to use the filter to retrieve all the access review series scoped to all Microsoft 365 groups in a tenant.

```pwsh
Import-Module Microsoft.Graph.Identity.Governance

Get-MgIdentityGovernanceAccessReviewDefinition -Filter "contains(scope/microsoft.graph.accessReviewQueryScope/query, './members')"
```

We're on the right track...

## Retrieving access definitions by types

So we can filter which type we're interested in.

```pwsh
function Get-AccessReviewDefinitionByFilter {
    [CmdletBinding()]
    param (
        [Parameter(ParameterSetName = 'GroupId')]
        [guid]$GroupId,

        [Parameter(ParameterSetName = 'Groups')]
        [switch]$Groups,

        [Parameter(ParameterSetName = 'Members')]
        [switch]$Members,

        [Parameter(ParameterSetName = 'AccessPackageAssignments')]
        [switch]$AccessPackageAssignments,

        [Parameter(ParameterSetName = 'RoleAssignmentScheduleInstances')]
        [switch]$RoleAssignmentScheduleInstances
    )

    switch ($PSCmdlet.ParameterSetName) {
        'GroupId'{
            Get-MgIdentityGovernanceAccessReviewDefinition -Filter "contains(scope/microsoft.graph.accessReviewQueryScope/query, '/groups/$($GroupId)')"
        }
        'Members'{
            Get-MgIdentityGovernanceAccessReviewDefinition -Filter "contains(scope/microsoft.graph.accessReviewQueryScope/query, './members')"
        }
        'AccessPackageAssignments'{
            Get-MgIdentityGovernanceAccessReviewDefinition -Filter "contains(scope/microsoft.graph.accessReviewQueryScope/query, 'accessPackageAssignments')"
        }
        'RoleAssignmentScheduleInstances'{
            Get-MgIdentityGovernanceAccessReviewDefinition -Filter "contains(scope/microsoft.graph.accessReviewQueryScope/query, 'roleAssignmentScheduleInstances')"
        }
        'Groups'{
            Get-MgIdentityGovernanceAccessReviewDefinition -Filter "contains(scope/microsoft.graph.accessReviewQueryScope/query, '/groups')"
        }
    }
}
```

This makes retrieving the access review definition a bit less confusing. We're using the _ParameterSetName_ to distinguish between the different types. It also prevents us from selecting more than one at the same time.

The _GroupId_ is a special one. If you're curious or want the access review of a specific security group just pass the objectId, If anything is found it'll return the accessReview(s). If it returns more than one, you should probably look into why that is. That could make the process of making decisions complicated down the line. I'm still learning, but this is something my colleague ran into. Here's how to use the functions

```pwsh
Get-AccessReviewDefinition -DisplayName 'automation-testing' #$null if not found
Get-AccessReviewDefinition -Id '4fa06f15-7cf9-437e-aa3f-aad8eeefb66e' #fails if it doesn't exist

#Filters return $null if they don't find anything. I feel this is more friendly.
Get-AccessReviewDefinitionByFilter -Groups
Get-AccessReviewDefinitionByFilter -GroupId '24354705-2d66-4a50-958b-99e4e30d90a5' #Pass the objectId of the security group
Get-AccessReviewDefinitionByFilter -Members
Get-AccessReviewDefinitionByFilter -AccessPackageAssignments
Get-AccessReviewDefinitionByFilter -RoleAssignmentScheduleInstances
```

## Work smarter not harder

We all know AI is a great personal assistant. I regulary fall back in routine of googling. Came across a great prompt to get a better grasp of AccessReview. Mind you this was all in hindsight, but as a reinforcement, quite impressive!

>"I want to learn about EntraId access review. Identify and share the most important 20% of learnings from this topic that will help me understand 80% of it."

Give it a try, It will not dissapoint! :wink:

Here are some points AI brought to my attention, great for selling an implementation to management... :wink:

### Initiating an Access Review

- Define Review Scope: Specify which resources and users will be reviewed.
- Select Reviewers: Assign individuals or groups who will conduct the review.
- Set Review Period: Determine the duration and frequency of the review process.

For security group access, I'd go with a monthly review. Access package review probably quaterly. PIM Roleassignments are tricky. Depending on how paranoid you are on a weekly basis, also maybe not for all roles. Some roles are more important than other... :wink: As long as you follow company's policies.

### Review Process

- Notification: Reviewers receive notifications to begin the review process.
- Review Actions: Reviewers approve, deny, or delegate access based on current needs and policies.
- Automated Recommendations: Leverage automated insights and recommendations to aid reviewers.
- Follow-Up: Post-review actions might include removing access for denied users or making necessary adjustments.

The recommendations are the chef's kiss! If a user hasn't logged in for quite some time it'll recommend removing the user. If you have an access package for this, they could always make a request in the future... Lifecycle management at its finest!

### Managing and Monitoring Reviews

- Dashboard: Use the Access Reviews dashboard to monitor ongoing and completed reviews.
- Reports: Generate and analyze reports to understand review outcomes and compliance.
- Audit Logs: Maintain logs of review actions for auditing and compliance purposes.

With all of the regulations and policies, audit logs & reports are a great provision.

### Benefits of Access Reviews

- Enhanced Security: Regular reviews help in identifying and mitigating unauthorized access.
- Compliance: Stay compliant with industry regulations and internal policies.
- Operational Efficiency: Streamline access management and reduce the burden on IT departments.

I can't tell you how many times I've heard the complaint "IT takes so long to grant me access" Now you get to delegate that responsibility... Yes Karen, now you can DIY... Oh and we have logs... So tread carefully... Hehe...

## Conclusion

This is just the tip of the iceberg! EntraId Identity governance will help organizations with keeping up with compliances, if implemented correctly. It isn't for the faint of heart, but the rewards are well worth it!

Ttyl,

Urv

## Additional resources

- [Governance](https://learn.microsoft.com/en-us/graph/api/resources/identitygovernance-overview?view=graph-rest-1.0)
- [Microsoft.Graph.Identity.Governance](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.governance/?view=graph-powershell-1.0)

## Appendix

[MSLearn Identit Governance](https://learn.microsoft.com/en-us/entra/id-governance/identity-governance-overview)
