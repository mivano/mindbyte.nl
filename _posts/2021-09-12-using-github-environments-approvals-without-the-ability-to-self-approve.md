---
published: true
title: Using GitHub Environments approvals without the ability to self approve
tags:
  - GitHub
  - Azure
  - CICD
  - Compliance
---

While developing code, you apply the best practices, like working from a short-lived feature branch and performing a pull request to the main branch when your feature is done. This allows for valuable feedback and makes sure that at least one other pair of eyes looks at your code, which can be a required item for compliance. Of course, you do need to enforce this with branch policies in GitHub.

This will make sure that code in the main branch is checked, but what about deployments? This might not be an issue for a test environment, but when deploying to production, you want to make sure you have more control over this. 

One way you can do this via GitHub is using the [Environments](https://docs.github.com/en/actions/reference/environments) feature. It contains a couple of interesting features:

1. Specify who can approve a deployment to this environment.
1. Which branch(es) can be used to deploy from.
1. What are the secrets for this environment.

Let's first look at number 2; this allows you to say that only code in the main branch (so which did have had pull requests and/or additional tests) can be used for this environment. Or specify that only branches which have protection rules enabled can be used. That is already an excellent filter to make sure we have code deployed that has gone through a defined process.

Next are the secrets; you do not want to have somebody create a new workflow and start using a repository secret that has access to production. This bypasses the Environments functionality entirely and gives you direct access to production. Using specific environments allows you to scope your secrets to the right environment.

And then number one; approvals for deployment. In my opinion, you want to bring your developed features to production as soon as possible so you can bring value to your end-users. All the checks and controls can be added in the Continuous Integration pipeline as well and are closer to the development cycle than adding it to the end. However, there are enterprises that require formal approvals before a release is deployed to a production environment. GitHub Environments covers you with an option to set a list of approvers. Before the release is deployed, the approvers will be asked to approve the deployment.

That all looks fine, but what if you have a devops team and are okay with somebody else in the team giving their approval? It works fine to add the team itself as the approver. The major drawback; you can approve your own release, something that auditors might actually not like.

The original [GitHub feature](https://github.com/github/roadmap/issues/167) did mention this specifically, but was not implemented:

> How will it work? Finally, organizations will be able to configure rules about approvals such as not allowing the actor that triggered the run to be the same actor that approves it. 

In Azure DevOps, this is an available feature; the user requesting the release, could not self-approve the release.

## Solution

So what to do when you do want to be compliant and still use environments and GitHub Actions? Well, what if we could find out who approved the release and check if that is the same person as the one who requested the release? If so, we fail the release. It might not be the nicest looking solution (no direct visibility in the UI, although it is in the logs), it does do the job of making sure no unauthorized access to an environment happens.

To set this up, you do need an environment and do need to set up approvers. This allows you to make sure nobody can access the secrets for production without somebody else being aware of it.

Inside the actual GitHub Actions workflow, you include the following `actions/github-script`:

```yml
- uses: actions/github-script@v4
  with:
    script: |
      
      // Get the approvals for this run
      const approvals = await github.request('GET /repos/{owner}/{repo}/actions/runs/{run_id}/approvals', {
        owner: context.repo.owner,
        repo: context.repo.repo,
        run_id: context.runId
      })

      // Empty check
      if (approvals.data.length == 0){
        core.setFailed(`There are no approvals for this deployment. This is not allowed.`);          
        return; 
      }
        
      // We are only interested in the latest one, which is the first one in the array 
      const latestApproval = approvals.data[0];
      if (latestApproval.user.login == context.actor){
        core.setFailed(`Deployment to the environment is approved by the same person (${context.actor}) who initiated the deployment. This is not allowed.`);            
        return;
      }
```

Basically, it requests a list of approvals for this run. If there are no approvals, it fails the release. If there is an approval (the latest), it checks if the actor is the same as the one who initiated the deployment. This will also work if you redeploy the same run.

Combining this with the complete workflow:

```yml
name: Release [main] to prd

# Controls when the action will run. 
on:
  release:
    types: [published]

jobs:
  deploy_prd:  
    name: Deploy to Production
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    # The environment to deploy to
    environment: 
      name: prd
    steps:
      - uses: actions/github-script@v4
        with:
          script: |
            
            // Get the approvals for this run
            const approvals = await github.request('GET /repos/{owner}/{repo}/actions/runs/{run_id}/approvals', {
              owner: context.repo.owner,
              repo: context.repo.repo,
              run_id: context.runId
            })
            // Empty check
            if (approvals.data.length == 0){
              core.setFailed(`There are no approvals for this deployment. This is not allowed.`);          
              return; 
            }
             
            // We are only interested in the latest one, which is the first one in the array 
            const latestApproval = approvals.data[0];
            if (latestApproval.user.login == context.actor){
              core.setFailed(`Deployment to the environment is approved by the same person (${context.actor}) who initiated the deployment. This is not allowed.`);            
              return;
            }

      # Login to Azure
      - name: Azure Login PRD.
        uses: azure/login@v1
        if: success()
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
```

For simplicity, I skipped the build and deploy steps. Note that you need to add an `if: success()` to the subsequent steps, so they only run when the check is correct. Also, see that the `prd` environment is requested. This will trigger the approval and provides access to the secrets in this environment.

Removing this step from the workflow file won't work as that would have been captured during a code review (the protected branches).

## Conclusion

Although I'm not too fond of the compliancy requirement that somebody else needs to approve a production release, we need to deal with it. I hope GitHub builds this in someday and brings it more in line with the feature set already in Azure DevOps, and then this solution is obsolete. But for now, it allows me to still use GitHub Actions for production releases.