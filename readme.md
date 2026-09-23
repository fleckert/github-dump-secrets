# github-dump-secrets

GitHub masks secrets within the GitHub workflow logs... and this is a good thing.

!!! Use the following snippets with caution !!!

## GitHub secret logging

To log a GitHub secret... [log-secret.yaml](./.github/workflows/log-secret.yaml)

```yaml
on:
  workflow_dispatch:

jobs:
  job:
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ secrets.github_token }}" | sed 's/./& /g'
```


## GitHub secret names logging

To log the names of the GitHub secrets... [log-secrets-names.yaml](./.github/workflows/log-secrets-names.yaml)

```yaml
on:
  workflow_dispatch:

jobs:
  job:
    runs-on: ubuntu-latest
    steps:
      - run: echo '${{ toJson(secrets) }}'
```


## GitHub secret names logging

To log all GitHub secrets... [generate-workflow.yaml](./.github/workflows/generate-workflow.yaml)
- resolve all GitHub secret names and
- create a new workflow file
- that mimics [log-secret.yaml](./.github/workflows/log-secret.yaml) for each GitHub secret name
- and... run the workflow.

A Pull request will be created that... logs all GitHub secrets.

Rquirememts:
- add a GitHub secret `WORKFLOW_PAT` with `workflow` permissions.

```yaml
on:
  workflow_dispatch:

jobs:
  job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          token: ${{ secrets.WORKFLOW_PAT }}
      
      - name: Generate Workflow
        env:
          GH_TOKEN: ${{ secrets.WORKFLOW_PAT }}
        run: |

          branchName="secrets-$(date +%s)"

          fileName="./.github/workflows/$branchName.yaml"
          touch $fileName

          echo 'on:'                                                                              >> $fileName
          echo '  push:'                                                                          >> $fileName
          echo '    branches:'                                                                    >> $fileName
          echo '      - '"$branchName"                                                            >> $fileName
          echo 'jobs:'                                                                            >> $fileName
          echo '  job:'                                                                           >> $fileName
          echo '    runs-on: ubuntu-latest'                                                       >> $fileName
          echo '    steps:'                                                                       >> $fileName
          echo '      - run: |'                                                                   >> $fileName
          names=($(echo '${{ toJson(secrets) }}' | jq -r 'keys[]'))
          s0="\$"
          s1="{"
          s2="}"
          for name in "${names[@]}"; do
            echo '            echo ------------------------------------'                          >> $fileName
            echo '            echo '"$name"                                                       >> $fileName
            echo '            echo '"$s0$s1$s1"' secrets.'"$name $s2$s2"' | sed '"'"'s/./& /g'"'" >> $fileName
          done
          
          echo '------------------------------------'
          cat $fileName
          echo '------------------------------------'

          git switch -c $branchName
          git config --global user.email "${{ github.actor }}@users.noreply.github.com"
          git config --global user.name "${{ github.actor }}"
          git add .
          git commit -m $branchName
          git push origin $branchName

          gh pr create --base main --head $branchName --title $branchName --body "See workflow changes"
```