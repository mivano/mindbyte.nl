---
published: true
title: Build a GitHub Actions Marketplace using the Github CLI, Jekyll and Tailwind CSS
tags:
  - GitHub
  - Actions
  - CLI
  - Tailwind
  - Jekyll
---

> TLDR; see the [repo](https://github.com/mivano/github-actions-catalog) or the [demo website](https://mivano.github.io/github-actions-catalog) for more details.

When you work with GitHub, you most likely use GitHub Actions as well. A great way to execute all kinds of tasks, like interacting with Git, performing a build, running tests, or deploying to an environment. 
There are a lot of different actions available since it is so easy to create one. With some metadata, you can add your actions to the [GitHub marketplace](https://github.com/marketplace?type=actions) for others to find.

That all looks good and nice, but that flexibility also comes with a price; security. As everybody can create actions, they can also add (not always on purpose) vulnerabilities to that action. You should check the actions source code to see if it is okay to use. And when you are using it, make sure to stick it to a version (like a sha) so that even if the code is changed, you still point to the version you vetted. Although this provides some protection, it might not be enough. My colleague Rob Bos wrote an excellent [blog](https://devopsjournal.io/blog/2021/10/14/GitHub-Actions-Internal-Marketplace) about this, highlighting all the risks and solutions. 
So if you follow his advice, you will end up with your own organization with GitHub Actions that you allow others to use. Discoverability comes into play now; how do people find which actions are available?

Rob has created a [GitHub Marketplace](https://github.com/rajbos/actions-marketplace) for this, and I took his idea to experiment with some other tools as well. I wanted a static site, so server-generated, using [Jekyll](https://jekyllrb.com/), which works great with GitHub pages. I also wanted to use [Tailwind CSS](https://tailwindcss.com/) to style it. A very flexible CSS framework. Next, the collection of actions needed to be simple, so I wanted to use the [GitHub CLI tool](https://github.com/cli/cli). This CLI allows you to perform all kinds of operations against the GitHub API.

## Collection GitHub actions

GitHub actions are nothing more than repos with an action YAML file inside. Simply put, we need to iterate through all the repositories in an organization and find the action file. 

There is a cli command called `gh repo list` that lists all the repositories in an organization. It retrieves only 30 repositories at a time, so you need to use the `--limit` option to get all the repositories. I ended up using the direct API call, which supports pagination.

So to fetch all repositories and loop through them, we can use this command and only return the name property.

```bash
repos=$(gh api /orgs/$org/repos --paginate --jq '.[].name')
for repo in $repos; do
  echo "Collecting $repo"
done
```

We now need to check if the file called `action.yml` actually exists by downloading it. The API option helps here again:

```bash
actionfilename="action.yml"
action=$(gh api -H 'accept: application/vnd.github.v3.raw' "repos/$org/$repo/contents/$actionfilename") || false
```

This command will directly read the raw contents of the file from Git. We need to handle the case when this file does not exist. It might be available under the `yaml` file extension, so we need to check for that.

```bash
if [ $? -ne 0 ]; then       
  actionfilename="action.yaml"
  action=$(gh api -H 'accept: application/vnd.github.v3.raw' "repos/$org/$repo/contents/$actionfilename") || false
fi
```

If the content is available, we can parse the YAML using the `shyaml` library.

```bash
name=$(echo "$action" | shyaml -q get-value name)
description=$(echo "$action" | shyaml -q get-value description)
author=$(echo "$action" | shyaml -q get-value author "")
```

We also want to have the readme file, so we can show that on the detail page. There is another API endpoint that retrieves this for us.

```bash
readme=$(gh api -H 'accept: application/vnd.github.v3.raw' "repos/$org/$repo/readme") || false
```

When we fork an existing repository, we do not get the releases, but do get the tags. They represent versions, so we also fetch those:

```bash
tags=$(gh api /repos/$org/$repo/git/refs/tags --paginate --jq .[].ref)
versions=""
for tag in $tags; do
  tag=$(echo $tag | cut -d/ -f3)
  versions+=" - $tag\n"     
done
```

The tags are in the form of `refs/tags/v1.0.0`, so we need to cut the `refs/tags/` part, and add them to a formatted string.

The last step is to save this to a markdown file, which Jekyll will render to a static page.

```bash
echo -e "---\nname: $name\ndescription: $description\nrepo: $org/$repo\nauthor: $author\nmetadata: $actionfilename\nversions:\n$versions---\n\n$readme" > $folder/$repo.md
```

You can find the complete file in the [GitHub repo](https://github.com/mivano/github-actions-catalog/blob/main/collect.sh).

## Showing the results

When the bash script has run, we will have a collection of markdown files inside the `_actions` folder. The workflow file (or when building locally) will convert those into html so they can be directly viewed in the browser. 
Since it is a Jekyll site, we can use templating and liquid to place the content elements in the right place. A demo of the result can be found [here](https://mivano.github.io/github-actions-catalog/).

## Conclusion

Using the GitHub CLI it is relatively simple to gather the data needed. The GitHub Actions workflow provides the credentials, and the CLI is already installed on the GitHub Build Agent. Hosting a static website on GitHub pages with Jekyll is also a simple task and offers a lot of flexibility. 