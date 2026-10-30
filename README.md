# Zenodo Action
This is a variant of [zenodo-release](https://github.com/rseng/zenodo-release) Copyright © 2022, Vanessa Sochat. This variant focuses on creating a Zenodo release when a GitHub release is published. There is an option to manually run the Action (a workflow_dispath) but should only be used in situations where the GHA failed to complete after publishing the release.

This GitHub Action will automatically publish a new archive/package or update an existing archive/package on Zenodo when a GitHub release is created, and without needing to enable admin webhooks. To get this working you will need to:

1. Log into Zenodo
2. Under your name -> Applications -> Developer Applications -> Personal Access Tokens, create a new token and choose both scopes for deposit
3. Add the token to your GitHub repository secrets as `ZENODO_TOKEN`. Settings > Secrets > Actions > Repository Secret
4. Create a .zenodo.json file (or CITATION.cff) in the root of your repository (see [template](.zenodo.json))
5. Add the example GitHub Action below to your GitHub repository in the `.github/workflows` directory before your first release.
6. Once the archive is created on Zenodo (or if the archive already exists), you'll want to update your action to include the **parent** DOI (in the form "##.####/zenodo.#####") as environment variable `ZENODOD_DOI`
## GitHub Action

```
name: Zenodo Release

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      release_tag:
        description: Tag of the published release to deploy
        required: true
        type: string

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      # Leave blank for the first deposit; set to the Zenodo parent/concept DOI afterward.
      ZENODO_DOI: ""

    steps:
    - uses: actions/checkout@v7
      with:
        ref: ${{ github.event_name == 'workflow_dispatch' && inputs.release_tag || github.event.release.tag_name }}

    - name: Run Zenodo Deploy
      if: github.event_name == 'release'
      uses: nmfs-opensci/zenodo-gha@main
      with:
        token: ${{ secrets.ZENODO_TOKEN }}
        zenodo_json: CITATION.cff
        doi: ${{ env.ZENODO_DOI }}

    - name: Get published release archive and metadata
      if: github.event_name == 'workflow_dispatch'
      id: release
      env:
        GH_TOKEN: ${{ github.token }}
        RELEASE_TAG: ${{ inputs.release_tag }}
      run: |
        release=$(gh api "repos/${GITHUB_REPOSITORY}/releases/tags/${RELEASE_TAG}")
        tag=$(jq -er '.tag_name | select(length > 0)' <<< "$release")
        archive_url=$(jq -er '.zipball_url | select(length > 0)' <<< "$release")
        html_url=$(jq -er '.html_url | select(length > 0)' <<< "$release")
        release_name=$(jq -r '.name // .tag_name' <<< "$release")
        release_body=$(jq -r '.body // ""' <<< "$release")

        curl --fail --location --retry 3 --silent --show-error \
          --header "Authorization: Bearer ${GH_TOKEN}" \
          --header "Accept: application/vnd.github+json" \
          "$archive_url" --output zenodo-release.zip

        printf '## Release Name: %s\n' "$release_name" > body.txt
        printf '%s\n' "$release_body" >> body.txt
        printf 'version=%s\nhtml_url=%s\n' "$tag" "$html_url" >> "$GITHUB_OUTPUT"

    - name: Check out Zenodo deployment script
      if: github.event_name == 'workflow_dispatch'
      uses: actions/checkout@v7
      with:
        repository: nmfs-opensci/zenodo-gha
        ref: main
        path: .github/zenodo-gha

    - name: Convert citation metadata and install deployment dependencies
      if: github.event_name == 'workflow_dispatch'
      run: |
        pip install --user cffconvert markdown requests
        cffconvert -f zenodo > zenodo.json

    - name: Deploy published release to Zenodo
      if: github.event_name == 'workflow_dispatch'
      env:
        VERSION: ${{ steps.release.outputs.version }}
        HTML_URL: ${{ steps.release.outputs.html_url }}
        ZENODO_TOKEN: ${{ secrets.ZENODO_TOKEN }}
        ZENODO_DOI: ${{ env.ZENODO_DOI }}
      run: |
        args=(zenodo-release.zip --version "$VERSION" --html-url "$HTML_URL" --zenodo-json zenodo.json)
        if [[ -n "$ZENODO_DOI" ]]; then
          args+=(--doi "$ZENODO_DOI")
        fi
        python .github/zenodo-gha/scripts/deploy.py upload "${args[@]}"
```

## Customizing your Zenodo entry infomation

This is taken from the `.zenodo.json` or `CITATION.cff` file. You need to fill these in with the author infomation. `CITATION.cff` is the better option because it can be used more widely, but these files are finicky so definitely set up your CITATION file using a cff validator. They are available in Python and R (e.g. `cffr` in R). Do a Google search. Note the the cff version (in the cff file) is important for the validator as versions 1.1, 1.2 and 1.3 have different schema (schema = names for the entries like 'author', 'title', etc).

## Troubleshooting Zenodo release fails

1) Go to Zenodo and make sure there is not a draft release. If there is, you will need to delete that.
2) Often a problem with the `.zenodo.json` file will cause failed releases. Go to the Actions tab, click on the failed action and look at the errors.
3) If you use CITAITON.cff to make the .zenodo.json file. If your CITATION.cff is not valid, then the Zenodo release will fail. Use a separate cff validator like `cffr` in R to check. cff files are very picky and it is easy to have validation issues.


