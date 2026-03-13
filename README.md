name: BugHunter Agent

on:
workflow_dispatch:
inputs:
target:
description: "zooplus.com"
required: true

jobs:
recon:
runs-on: ubuntu-latest

steps:
  - name: Install Tools
    run: |
      sudo apt update
      sudo apt install -y jq git python3-pip
      pip3 install truffleHog waybackurls gau

  - name: GitHub Repo Discovery
    run: |
      TARGET=${{ github.event.inputs.target }}
      curl -s "https://api.github.com/search/repositories?q=$TARGET" | jq -r '.items[].clone_url' > repos.txt

  - name: Clone Repos
    run: |
      mkdir repos
      while read repo; do
        git clone --depth=1 $repo repos/ || true
      done < repos.txt

  - name: Secrets Scan
    run: |
      trufflehog filesystem repos/ > secrets.txt || true

  - name: Endpoint Extraction
    run: |
      grep -RhoP '(http|https)://[^"]+' repos/ > endpoints.txt || true

  - name: Archive Discovery
    run: |
      find repos -type f -name "*.zip" -o -name "*.tar" -o -name "*.sql" -o -name ".env*" > archives.txt

  - name: Subdomain Clues
    run: |
      grep -RhoP '([a-zA-Z0-9_-]+\.)+'$TARGET repos/ | sort -u > subdomains.txt || true

  - name: Report
    run: |
      cat secrets.txt endpoints.txt archives.txt subdomains.txt > final-report.txt

  - name: Upload Report
    uses: actions/upload-artifact@v4
    with:
      name: bughunter-report
      path: final-report.txt
