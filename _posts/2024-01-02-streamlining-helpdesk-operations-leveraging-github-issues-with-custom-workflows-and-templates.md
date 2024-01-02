---
published: 2024-01-02T20:59:57.060Z
title: "Streamlining Helpdesk Operations: Leveraging GitHub Issues with Custom Workflows and Templates"
tags: [github, github-actions, helpdesk]
header:
  teaser: "https://mindbyte.nl/images/cdc-_XLJy3h77cw-unsplash.jpg"
slug: streamlining-helpdesk-operations-leveraging-github-issues-custom-workflows-templates
---

## Setting the Scene: Choosing GitHub Issues for Internal Helpdesk Needs.

In the realm of helpdesk and ticketing systems, the market is brimming with options. From Zohodesk and Freshdesk to Zendesk and Jira, the choices are many, some even offering enticing free entry points. However, a common catch with these platforms is the pricing model, which typically scales based on the number of agents, leading to a significant cost increase over time. This challenge became evident in a recent customer project I was involved in. While we successfully implemented Freshdesk as the support system for the customer's SaaS application, our internal requirements painted a different picture.

Our needs revolved around internal operations like account management, onboarding/offboarding procedures, addressing hardware issues, and managing licenses—tasks distinctly separate from customer interactions and not fitting neatly into the development team's responsibilities. Adding to the complexity was the customer's ISO certification, necessitating meticulous tracking of changes and actions. This scenario required a separate system from the customer-facing helpdesk, yet something that was both minimalistic and easy to manage.

The solution lay closer than we thought. With all team members already well-versed in GitHub, leveraging its environment was a logical step. GitHub Issues, when combined with customized workflows and templates, presented a unique opportunity. It offered a familiar platform that was both cost-effective and adaptable to our specific internal needs. In this post, I'll guide you through the process of setting up GitHub Issues as an efficient, internal helpdesk system, elucidating how it can be tailored to streamline your organizational processes, just as it did for ours.

## Laying the Foundations: Initial Setup of Your GitHub Helpdesk

In the journey of transforming GitHub into an effective internal helpdesk, the initial setup plays a crucial role. GitHub offers two distinct systems for tracking activities: Issues and Discussions. Understanding the nuances of each is key to leveraging them effectively.

### Issues vs. Discussions

- **Issues**: This feature in GitHub is straightforward. Each issue comprises a title, a descriptive body, optional labels, and assignments. It's the go-to choice for tracking specific tasks, bugs, or requests.
- **Discussions**: In contrast, Discussions provide a forum-like environment. They're ideal for cases where you want to engage in a broader conversation before possibly converting the discussion into a more formal issue. Not everything that crops up is immediately an issue, and Discussions provide the flexibility to explore topics more broadly.

For the purpose of an internal helpdesk, where submissions are likely to be direct issues, focusing on the Issues feature is the most practical approach.

## Setting Up Your Repository

1. **Create a New Repository**: Start with a fresh slate by setting up a new repository within your organization. A simple and descriptive name like 'servicedesk' or 'helpdesk' works well. Remember to keep it internal/private to maintain confidentiality. ![Create Repository](/images/create-helpdesk-repo.png)
2. **Craft Your README.md**: The `README.md` file is the first point of contact for users visiting your repository. It’s a Markdown file that can be used to provide essential instructions and guidelines. Here’s how you can structure it:

*Introduction*: Briefly describe the purpose of the helpdesk and what types of issues should be submitted here.

*How to Submit an Issue*: Give clear, step-by-step instructions on how to create a new issue, including guidance on writing a good title and description.

*Label Usage*: Explain how labels are used within the repository to categorize and prioritize issues.

*Response Times and Process*: Outline what users can expect in terms of response times and the process their issues will go through.

For example, here's a sample README.md file:

```markdown
# Service Desk

Service Desk system for X.

## What is this?

This repository can be used to record issue requests to the service organisation operated by Y. Like requesting access to or gaining new accounts, having issues with hardware, or have problems with your workstation or phone.

This is not for software production issues; that is for the devteam, reachable in [Slack](link to slack).

Customers having questions or problems with the usage of Z, will need to go to [Freshdesk](link to freshdesk).

## How does it work?

When you want to register an item, create a [new issue](https://github.com/yourorg/yourrepo/issues/new/choose) and use the most applicable template. 

The issue will be automatically assigned to the service team at Y and they will react via the issue. 

```

Creating this foundational setup ensures a streamlined experience for both those managing the helpdesk and those using it, setting the stage for efficient internal support management.

## Optimizing Issue Reporting: Implementing Issue Templates

A common challenge in issue tracking is receiving submissions that lack essential information. This often leads to a back-and-forth to gather the necessary details, causing delays and inefficiencies. To address this, GitHub offers a powerful tool: issue templates.

### Creating Issue Templates

1. **Template Files**: Start by creating your issue templates. These templates are files that reside in the `.github/ISSUE_TEMPLATE` folder of your repository. You can create multiple files here to cater to different types of requests or issues.

2. **Design Your Templates**: Each template should be designed to collect specific data pertinent to the type of issue it represents. This structured approach ensures that when a user submits an issue, they provide all the necessary information upfront.

An example of a template for requesting a new account:

```yaml
name: Accounts
description: Onboarding or offboarding, getting access to a system
title: "[Account]: "
labels: ["account", "triage"]
assignees:
  - userx
  - usery
body:
  - type: markdown
    attributes:
      value: |
        Use this template to request access to a system, onboard or offboard an employee or ask for a change in permissions. 
  - type: input
    id: person
    attributes:
      label: Contact Details
      description: For who is the change intended?
      placeholder: ex. email@example.net
    validations:
      required: true
  - type: textarea
    id: what-need-to-change
    attributes:
      label: What need to change?
      description: What is the action that needs to be taken?
      placeholder: Tell us what needs to be done
      value: "This person need access to..."
    validations:
      required: true
  - type: input
    id: when
    attributes:
      label: When
      description: When does it need to be done?
      placeholder: Leave empty for ASAP or specify a date
    validations:
      required: false
  
```

Or for hardware issues:

```yaml
name: Hardware
description: Issues with hardware, like replacing, defects
title: "[Hardware]: "
labels: ["hardware", "triage"]
assignees:
  - userx
  - usery
body:
  - type: markdown
    attributes:
      value: |
        Use this template to request support on anykind of hardware you have; broken phone, PC not working, lost mouse, use this to get us involved.
  - type: input
    id: person
    attributes:
      label: Contact Details
      description: Who has issues with the hardware
      placeholder: ex. email@example.net
    validations:
      required: true

  - type: dropdown
    id: area
    attributes:
      label: Area
      options:
        - Desktop
        - Phone  
      description: Indicate which area is impacted 
  - type: textarea
    id: what-need-to-change
    attributes:
      label: What need to change?
      description: What is the action that needs to be taken?
      placeholder: Tell us what needs to be done
      value: "This piece of equipement is no longer working..."
    validations:
      required: true    
  - type: input
    id: when
    attributes:
      label: When
      description: When does it need to be done?
      placeholder: Leave empty for ASAP or specify a date
    validations:
      required: false
  ```

**Configuring the Template Selector**

**Create a config.yml File**: This file goes in the same `.github/ISSUE_TEMPLATE` folder. It's used to configure how users select an issue template.
For example, here's a sample config.yml file:

```yaml
blank_issues_enabled: true
contact_links:
  - name: Freshdesk
    url: https://support.x.nl/a/dashboard/default
    about: For customer issues.
```

**Configuration Options**: 
- `blank_issues_enabled: true` - This setting allows users to still create blank issues. However, disabling this feature means users can only use your predefined templates.
- **Contact Links**: These are additional options you can provide for users. For example:
  - Freshdesk for customer issues with a direct link to your Freshdesk dashboard.
  - Slack for devteam issues with a direct link to your Slack workspace.

   These links not only guide users to the right resources but also help in segregating different types of issues efficiently.

By implementing these templates and configuration settings, you streamline the issue submission process. Users are guided to provide all necessary information, reducing the need for follow-up and speeding up the resolution process. To direct users to the template selector, use a URL like `https://github.com/yourorg/yourrepo/issues/new/choose`. This ensures a more organized and efficient approach to managing internal helpdesk requests.

![Select an issue template](/images/select-issue-template.png)

## Effective Issue Management: Workflows and Automation

As the number of issues in your GitHub helpdesk repository grows, effective management and maintenance become crucial. To ensure efficiency and order, we utilize a combination of GitHub workflows and Probot actions.

### Preventing Empty Issues

**Request-Info Bot**: One of the common frustrations in issue tracking is receiving issues with a title but no descriptive content. To address this, we implement the Request-Info bot ([Request Info Probot](https://probot.github.io/apps/request-info/)). This bot is configured to prompt for more information on such issues. Install it in your helpdesk repository and create a `.github/config.yml` file with the following contents:

```yaml
# Configuration for request-info - https://github.com/behaviorbot/request-info

# *Required* Comment to reply with
requestInfoReplyComment: >
  We would appreciate it if you could provide us with more info about this issue!

# *OPTIONAL* default titles to check against for lack of descriptiveness
# MUST BE ALL LOWERCASE
requestInfoDefaultTitles:
  - update readme.md
  - updates

# *OPTIONAL* Label to be added to Issues and Pull Requests with insufficient information given
requestInfoLabelToAdd: needs-more-info
```

### Automated Assignment of Issues

**Auto-Assign Issue Action**: To ensure issues are promptly addressed, we use the [Auto-Assign Issue Action](https://github.com/marketplace/actions/auto-assign-issue). This action automatically assigns new issues to the responsible team members. While issue templates allow for assigning users, this action covers scenarios where blank issues are created.

### Labeling for Triage

**Triage-New-Issues App**: Apply a 'Triage' label to new issues using the [Triage New Issues App](https://github.com/apps/triage-new-issues). This helps in quickly identifying and categorizing new submissions for further action.

### Managing Stale Issues

**Stale Issue Workflow**: Keeping issues open for too long without activity can clutter your helpdesk. To manage this, we implement a workflow to mark stale issues:
   ```yaml
   name: 'Close stale issues and PRs'
   on:
     schedule:
       - cron: '30 1 * * *'

   jobs:
     stale:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/stale@v9
           with:
             stale-issue-message: 'This issue is stale because it has been open 30 days with no activity. Remove stale label or comment or this will be closed in 5 days.'
             days-before-stale: 30
             days-before-close: 5
   ```
   This workflow runs daily, identifying issues open for more than 30 days and marking them as stale. This process helps maintain a cleaner, more manageable issue list and ensures issues don't linger unresolved.

By integrating these tools and workflows, you create a dynamic and responsive helpdesk system within GitHub, capable of handling issues efficiently and ensuring nothing falls through the cracks.

## Harnessing Data: Implementing Analytics in Your GitHub Helpdesk

One of the advantages of professional helpdesk systems is their capability to provide analytics, such as resolution times and the duration for which issues remain open. While GitHub might not offer these features out-of-the-box, we can achieve similar analytics through a custom workflow.

### Setting Up Monthly Issue Metrics

This workflow is designed to run monthly and collect data on issues created and resolved within that period, providing valuable insights into the performance of your helpdesk system.

**Workflow Configuration**:
{% raw %}
```yaml
name: Monthly issue metrics
on:
  workflow_dispatch:
  schedule:
    - cron: '3 2 1 * *'

jobs:
  build:
    name: issue metrics
    runs-on: ubuntu-latest
    permissions:
      actions: read
      issues: write
    steps:

    - name: Get dates for last month
      shell: bash
      run: |
        # Script to calculate date range of the previous month
        [Date calculation script here]

    - name: Run issue-metrics tool
      uses: github/issue-metrics@v2
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SEARCH_QUERY: 'repo:yourrepo/servicedesk is:issue created:${{ env.last_month }} -reason:"not planned"'

    - name: Create issue
      uses: peter-evans/create-issue-from-file@v4
      with:
        title: Monthly issue metrics report
        content-filepath: ./issue_metrics.md
        assignees: mivano
```
{% endraw %}
   
**Key Components**:
- **Date Calculation**: The workflow begins by calculating the date range for the previous month. This range is used to filter the issues for the metrics report.
- **Issue Metrics Tool**: Utilizing the 'github/issue-metrics@v2' action, the workflow gathers data on issues created within the specified date range.
- **Report Creation**: An issue is automatically created containing the metrics, which can then be labeled, assigned, and reviewed like any other issue.

This workflow transforms GitHub into a more robust helpdesk tool, providing monthly analytics similar to those offered by specialized helpdesk systems. The generated report gives a clear picture of helpdesk performance, helping identify areas for improvement. More information on this action and its capabilities can be found on the [GitHub Blog](https://github.blog/2023-07-19-metrics-for-issues-pull-requests-and-discussions/). By incorporating this workflow, you enhance the functionality of your GitHub helpdesk, leveraging data to drive efficiency and effectiveness.

## Automating Recurring Tasks: Workflow for Scheduled Issues

In the management of any helpdesk or support system, certain tasks are bound to recur regularly. These can range from routine checks to periodic backups. To handle such recurring issues efficiently within GitHub, we can leverage the power of automated workflows. This approach ensures that these tasks are consistently addressed without fail.

### Creating a Workflow for Recurring Issues

Here's an example of a workflow designed to create a new issue for quarterly checks of users and authorizations:

**Workflow Definition**:
```yaml
name: Quarterly check users and authorisations
on:
  workflow_dispatch:    
  schedule:
    - cron: 0 12 1 */3 *

jobs:
  create_issue:
    name: Create issue - Quarterly check users and authorisations
    runs-on: ubuntu-latest
    steps:
      - name: Create issue 
        run: |
          new_issue_url=$(gh issue create \
            --title "$TITLE" \
            --label "$LABELS" \
            --body "$BODY" \
            --assignee "$ASSIGNEE")
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GH_REPO: ${{ github.repository }}
          TITLE: Quarterly check users and authorisations
          LABELS: iso
          ASSIGNEE: userX,userY
          BODY: |
            Perform the quarterly checks for users and the authorisations. See the ISMS for more details.
          PINNED: false
```

**Key Components**:
- **Cron Schedule**: Set to trigger quarterly (`0 12 1 */3 *`). This timing can be adjusted based on the frequency needed for the particular task.
- **Issue Creation Step**: Automates the creation of a new GitHub issue with specific details like title, labels, body content, and assignees.

### Expanding the Workflow

The beauty of this system lies in its flexibility. By modifying the trigger and the content of the issue, and saving it as a new file in the `.github/workflows` folder, you can create workflows for various other recurring issues. This method is not only efficient but also ensures consistency and reliability in performing routine tasks that are crucial for your organization.

Implementing such automated workflows for recurring issues in your GitHub helpdesk repository aids in maintaining regularity and ensures that important tasks are not overlooked. This level of automation brings a new dimension of efficiency to your internal helpdesk operations.

## Conclusion: GitHub as an Ideal Internal Helpdesk Solution

In conclusion, leveraging GitHub for your internal helpdesk system offers numerous advantages, especially when considering the ecosystem and tooling already familiar to many teams. This approach not only streamlines internal support processes but also adds a layer of efficiency and integration that traditional helpdesk tools may not provide.

### Key Advantages of Using GitHub for Internal Helpdesk:

1. **Unified Ecosystem**: Staying within GitHub means no need to juggle between different platforms, reducing the learning curve and integration complexities.

2. **Cost-Effective**: Eliminates the additional costs associated with third-party helpdesk tools, as GitHub already forms part of many organizations' toolsets.

3. **Robust Issue Tracking Features**: Utilize GitHub's native issue features like templates, formatting options, attachments, mentions, and cross-referencing, enhancing communication and tracking efficiency.

4. **Customizable Workflows and Apps**: The ability to add workflows and apps further tailors the experience to your organization's specific needs.

5. **Project Boards for Visualization**: Employ GitHub's project boards for a visual representation of tasks, allowing you to effortlessly manage and move items through to completion.

### External Use Considerations:

While GitHub excels for internal use, there are challenges when considering external helpdesk applications. Non-GitHub users cannot create issues, and external access to a private helpdesk repository is not advisable. Additionally, GitHub doesn't natively handle email-based ticket creation or manage attachments from external sources effectively.

### Solution for External Use: Scitor.io

For those seeking to extend GitHub's capabilities to an external audience, [Scitor.io](https://www.scitor.io) offers a solution. It transforms GitHub into a more traditional helpdesk system, bridging the gap for external user interactions.

### Wrapping Up

For internal team use, GitHub stands out as a practical, cost-effective, and efficient tool for helpdesk management. It consolidates various aspects of issue tracking and resolution into a single, integrated platform, familiar to most developers and IT professionals. By following the outlined steps and considerations, you can effectively transform GitHub into a powerful tool for managing your internal helpdesk needs, harnessing its full potential to streamline and enhance your support processes.

> Photo by <a href="https://unsplash.com/@cdc?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">CDC</a> on <a href="https://unsplash.com/photos/man-in-black-and-white-checkered-dress-shirt-using-computer-_XLJy3h77cw?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>
  