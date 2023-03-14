---
published: true
title: Ensuring API Compatibility with OpenAPI Comparator and GitHub Workflows
tags:
  - API
  - REST
  - OpenAPI
  - GitHub
---

Maintaining backward compatibility is crucial when developing APIs. As APIs evolve, changes may be necessary to introduce new functionality or improve performance. However, these changes can also introduce breaking changes, making the API incompatible with existing applications that rely on it. This is why it's essential to be able to detect any breaking changes to an API and make sure that any changes made are backward compatible.

One way to detect breaking changes in APIs is by using open-source tools such as [OpenAPI Comparator](https://github.com/criteo/openapi-comparator) and [OpenAPI-diff](https://github.com/OpenAPITools/openapi-diff). OpenAPI Comparator is a tool developed by Criteo that can help detect breaking changes in an API by comparing different versions of the API specifications. OpenAPI-diff, on the other hand, is an open-source tool developed by OpenAPITools that can also compare different versions of an API specification and highlight any differences between them.

## Sample

To further illustrate the use of open-source tools such as OpenAPI-diff in detecting breaking changes in APIs, let's consider an example where we download the default Petstore OpenAPI definition file, make a breaking change, and run the OpenAPI-diff container to compare the old and new versions.

First, let's download the default Petstore OpenAPI definition file and save it in a folder:

```bash
mkdir petstore
cd petstore
curl https://raw.githubusercontent.com/OAI/OpenAPI-Specification/master/examples/v3.0/petstore.json -o petstore_old.json
```

Next, let's make a breaking change to the API by removing the delete operation from the Pet resource.
We can now save the new version of the API specification in a file called petstore_new.yaml.

Finally, we can run the OpenAPI-diff container using the following command:

```bash
docker run --rm -t \
  -v $(pwd):/specs:ro \
  openapitools/openapi-diff:latest /specs/petstore_old.json /specs/petstore_new.json
```

This command will compare the old and new versions of the Petstore API specification and output any breaking changes detected. In this case, the command will output a message indicating that the delete operation has been removed, which is a breaking change.

## Workflow

Using these tools, developers can compare the old and new versions of an API specification and detect any breaking changes. The comparison results can be outputted in different formats such as markdown, HTML, or JSON. This can be particularly useful in a Pull Request build process, where the comparison can be included as a step to ensure that any changes made to the API are backward compatible. If the comparison detects any breaking changes, the build can fail, preventing any backward-incompatible changes from being merged into the main branch.

Let's say we have a Node.js API project with an OpenAPI specification file located at openapi.yaml. We want to make sure that any pull requests made to this project do not introduce breaking changes to the OpenAPI specification.

To do this, we can create a GitHub workflow that runs the openapi-diff tool to compare the OpenAPI files in the pull request against the OpenAPI file in the main branch. Here's an example workflow file:

```yaml
name: OpenAPI Diff

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '14.x'

      - name: Install dependencies
        run: npm install

      - name: Generate OpenAPI file
        run: npm run generate-openapi

      - name: Check out BASE rev
        uses: actions/checkout@v2
        with:
          ref: ${{ github.base_ref }}
          path: base   

      - name: Compare OpenAPI files
        uses: docker://openapitools/openapi-diff:latest
        with:
          args: --fail-on-incompatible /github/workspace/openapi.yaml  base/schemas/openapi.yaml
```

Another use case for these tools is to validate the API specifications of third-party dependencies. By downloading the API specification of third-party dependencies and comparing it to the known version stored by the developer, any changes made by the third-party dependencies can be detected. This can be particularly useful in a nightly workflow, where the API specifications of third-party dependencies can be checked regularly to ensure that they are still compatible.

## Conclusion

In conclusion, maintaining backward compatibility is essential when developing APIs. Open-source tools such as OpenAPI Comparator and OpenAPI-diff can help detect any breaking changes in an API and ensure that any changes made are backward compatible. By using these tools, developers can improve the quality of their APIs and ensure that they are compatible with existing applications, preventing any disruptions or compatibility issues.
