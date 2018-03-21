# Integrate and change a VSTS pull request
:hp-tags: vsts
:hp-image: https://mindbyte.nl/images/vstsprstatus.png
:published_at: 2017-11-04

You might have seen it with GitHub; when you do a pull request there will most likely be a build being kicked off and that influences the state of the pull request. A failed build (or any other check) is shown on the https://developer.github.com/v3/guides/building-a-ci-server/[Pull Request page].

image::githubprstatus.png[]

Visual Studio Team System can do the same; you can define a build that needs to be successful before the Pull Request can be completed. You also can indicate who needs to approve the PR and even that all the remarks need to be resolved. However, you can also do some more interesting tricks by adding additional services that interface with the Pull Request. Such a service allows you to intercept the changes, determine if the PR can continue and block the PR when not allowed to be merged back into the target branch.

For example to check if the code applies to certain rules and conventions, if it has passed a certain qualification from an external system or contains a standard license header.

There is a good tutorial from https://docs.microsoft.com/en-us/vsts/git/how-to/create-pr-status-server[Microsoft] on how to build a nodejs application to do this so I won't repeat that here. Interesting is that you can use https://ngrok.com/download[ngrok] to map a public domain name to a local port so you can test this from your own machine. 

## Comments

Besides changing the state of the PR (and make it blocked), you can also output comments to the PR. You can do this on individual files and even blocks of code in the file.

Creating a comment is exposed by the vsts node library:

```javascript
var comments # {
        "comments": [
        {
            "content": "I like your *code* style!",
            "commentType" : 1,
            "parentCommentId" : 0
        }], 
        "properties": {
            "Microsoft.TeamFoundation.Discussion.SupportsMarkdown": {
                "type": "System.Int32",
                "value": 1
            }
        },
        "status": 1
    }

    vstsGit.createComment(comments, repoId, pullRequestId, 0, projectId).then( result #> {
        console.log(result);
    }).catch(result #> {
        console.log(result);
    })
```

Best is to check with the documentation https://docs.microsoft.com/en-us/rest/api/vsts/git/pull%20request%20threads/create[here] as the node library is not up to date.

When you create a PR and have your application configured under service hooks, you will get the following in your PR screen:

image::vstsprstatus.png[]

## Policy

When the service hook is defined as a PR integration, it will show up in the branch policies page as an exteral service. 

image::vstsbranchpolicy.png[]

Here you can also make this a mandatory rule. So it has to be approved before it can be merged. If you try to complete a PR and the condition did not apply, you will get a nice warning.

image::vstsprrejected.png[]

## Code

I placed the code on https://github.com/mivano/pr-status[GitHub]; this contains some simple logic to mark a PR to *pending* when you include a _WIP_ in the title or mark it as *error* when it sees an _error_ word in the text. These are just examples, you would check if it complies with your rules. It might need to have a correct description or certain artifacts.

The code will also add a comment to the PR. You can make threaded conversations, set the state and even annotate files.

You need to set your own Personal Access Token and VSTS address. I used a .env file which you can put in the same folder as your app.js.

```
COLLECTIONURL#https://address.visualstudio.com
TOKEN#yourtoken
```





