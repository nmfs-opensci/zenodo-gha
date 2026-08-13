# Zenodo Action
This is a variant of [zenodo-release](https://github.com/rseng/zenodo-release) Copyright © 2022, Vanessa Sochat. This variant is focused specifically on creating a Zenodo release when a GitHub release is made so I moved some of set-up into `action.yml` because this is only triggered by GitHub releases.

This is a GitHub Action to allow you to automatically update a package on Zenodo
when a GitHub release is created, and without needing to enable admin webhooks. To get this working you will need to:

1. Log into Zenodo
2. Under your name -> Applications -> Developer Applications -> Personal Access Tokens, create a new token and choose both scopes for deposit
3. Add the token to your GitHub repository secrets as `ZENODO_TOKEN`. Settings > Secrets > Actions > Repository Secret
4. Create a .zenodo.json file (or CITATION.cff) in the root of your repository (see [template](.zenodo.json))
5. Add the example GitHub Action below to your GitHub repository in the `.github/workflows` directory before your first release and use the 'New Release' instructions. Alternatively, create a release manually on Zenodo and use the 'Existing Release' instructions.

## New Release (no DOI)

If you have no entry on Zenodo yet, then create a `zenodo.yml` file in the `.github/workflows` directory and copy this code into that file. Then make a GitHub release to trigger the GitHub Action. You can see if it worked by going to the Actions tab. Then go to Zenodo and look up what DOI it created. Copy the parent DOI (all versions DOI).

```
name: Zenodo Release

on:
  release:
    types: [published]

jobs:
  deploy:
    runs-on: ubuntu-20.04

    steps:
    - uses: actions/checkout@v3
    - name: Run Zenodo Deploy
      uses: nmfs-opensci/zenodo-gha@main
      with:
        token: ${{ secrets.ZENODO_TOKEN }}
        zenodo_json: .zenodo.json   # either a .zenodo.json file or CITATION.cff
```

## Existing Release (has DOI on Zenodo)

Edit your `.github/workflows/zenodo.yml` file and add the `doi: ` line. Go to Zenodo to your archive for the your first release (above). Make sure it is the parent DOI shown on Zenodo called the "cite all versions by using the DOI". Now when you make releases, they will be added to the Zenodo record.

```
name: Zenodo Release

on:
  release:
    types: [published]

jobs:
  deploy:
    runs-on: ubuntu-20.04

    steps:
    - uses: actions/checkout@v3
    - name: Run Zenodo Deploy
      uses: nmfs-opensci/zenodo-gha@main
      with:
        token: ${{ secrets.ZENODO_TOKEN }}
        zenodo_json: .zenodo.json   # either a .zenodo.json file or CITATION.cff
        doi: 10.5281/zenodo.10811321 # update with your DOI
```

## Customizing your Zenodo entry infomation

This is taken from the `.zenodo.json` or `CITATION.cff` file. You need to fill these in with the author infomation. `CITATION.cff` is the better option because it can be used more widely, but these files are finicky so definitely set up your CITATION file using a cff validator. They are available in Python and R (e.g. `cffr` in R). Do a Google search. Note the the cff version (in the cff file) is important for the validator as versions 1.1, 1.2 and 1.3 have different schema (schema = names for the entries like 'author', 'title', etc).

## Troubleshooting Zenodo release fails

1) Go to Zenodo and make sure there is not a draft release. If there is, you will need to delete that.
2) Often a problem with the `.zenodo.json` file will cause failed releases. Go to the Actions tab, click on the failed action and look at the errors.
3) If you use CITAITON.cff to make the .zenodo.json file. If your CITATION.cff is not valid, then the Zenodo release will fail. Use a separate cff validator like `cffr` in R to check. cff files are very picky and it is easy to have validation issues.


