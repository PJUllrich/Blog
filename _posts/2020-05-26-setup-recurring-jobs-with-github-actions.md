---
layout: post
title: How to set up reoccurring jobs with GitHub Actions 
description: How to automate a reoccurring job using GitHub Actions in order to fetch and upload latest data to a GitHub repository.
date:   2020-05-26 13:01:35 +0200
tags:   [development, github]
---

In a personal project of mine, I regularly fetched data collected by bicycle counting stations scattered around Cologne, Germany, and stored the data in a public [GitHub repository](https://github.com/PJUllrich/Dauerzaehlstellen-Koeln/). I ran the Python script which was fetching, appending, and uploading the data to GitHub manually for a while, but since I (am lazy and) wanted to always offer the latest data, I thought about how I could automate this process. Here’s how I accomplished running the reoccurring process with GitHub Actions.

At first, I thought about using the popular [Oban](https://github.com/sorentwo/oban) library written in Elixir to run a reoccurring job, but every time the job would have been run, I would have to clone the repository, fetch the data, commit the changes and push everything to GitHub using a personal access token. Since I could not use git commands directly from Elixir, I would also have to use a wrapper library like [Xgit](https://github.com/elixir-git/xgit). This seemed way too complicated and the friendly Elixir community [pointed me](https://elixirforum.com/t/how-to-update-a-github-repo-daily/31798/3?u=pjullrich) towards using `scheduled` GitHub Actions.

In short: GitHub Actions are jobs, scripts, or routines which GitHub runs for you whenever a predefined event occurs. Most people use GitHub Actions to automate their Continuous Integration (CI) pipeline using push or `pull_request` triggers. This means that whenever somebody pushes changes to a GitHub repository or opens a Pull Request, the GitHub Action is executed, mostly to run some tests or to upload the latest version to a package manager. What I was unaware of was the [multitude of triggers](https://help.github.com/en/actions/reference/events-that-trigger-workflows) that can cause a GitHub Action to be run.

In my case, the [scheduled](https://help.github.com/en/actions/reference/events-that-trigger-workflows#scheduled-events-schedule) trigger seemed most interesting. It can be set up using (relatively) simple `cron` syntax, which would let me trigger a GitHub Action once a day to fetch and update the data. Long story short, here’s my GitHub Action workflow, which I shortened a bit from the [original](https://github.com/PJUllrich/Dauerzaehlstellen-Koeln/blob/master/.github/workflows/python.yml), but it essentially achieves the same goal.

```yaml
name: Update data

on:
  schedule:
    - cron: "0 5 * * *"

jobs:
  execute:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Update local files
        run: python3 fetch_current_data.py

      - name: Commit files
        run: |
          git config --local user.email "your@email.com"
          git config --local user.name "GitHub Action"
          git commit -m "Update data" -a
          
      - name: Push changes
        uses: ad-m/github-push-action@master
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

This workflow is executed every day roughly at 5 am, give or take 15min since GitHub Actions aren’t run exactly on time. It clones the repository to its machine and executes a python script that fetches and appends the latest counts to local files holding the historical data. It then uses the [GitHub Push Action](https://github.com/ad-m/github-push-action) to automatically commit and push the latest counter data back to the `master` branch of my GitHub repository.

Using this GitHub Action, I could safely forget about running the Python script manually every day and it would automatically fetch and upload the latest data. So, if you also want to safely forget about things, use GitHub Actions.

## Caveats
As mentioned before, the Action is **not** executed exactly on time, but with a delay of up to 15min in my experience. This means, that you shouldn’t use scheduled GitHub Actions for reoccurring jobs which are time-sensitive.

I was not able to reproduce this since, but the GitHub Action **could** execute twice, so make sure that your job is idempotent or no-op on the second run. This means that you should be able to run your job multiple times and always receive the expected result. I achieved this by storing the latest date for which I fetched the data. Now, whenever the job was run again for the same date, it would finish without changing any data.

## Conclusion
It took me a day to research this possibility, but only around 1 hour to set up this GitHub Action, so GitHub Actions have convinced me once more of their ease-to-use and practicality. I hope this helped you further as well!