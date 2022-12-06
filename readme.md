# gh-repo-api-playground

This was a playground for me to try to use Github's git rest api to programmatically interact with git objects.

The idea was to manually push a commit through HTTP calls, and then amend that commit with a different file change.
The general approach is to 
* create a blob (for every changed file)
* create a tree that references the blobs for each file, and the tree of the parent commit (can also inline text instead of blob references; Also I did not reference a parent tree, which creates a clean repo structure for the commit)
* create a commit that references the tree
* update the reference of the main branch to point to the new commit

Code snippets below are Powershell-style formatted scripts that invoke Github's CLI tool, which wraps the REST API and includes authentication by itself. 

Useful links

* https://cli.github.com/manual/gh_api
* https://docs.github.com/en/rest/git/blobs?apiVersion=2022-11-28#create-a-blob
* https://github.com/settings/repositories

# Approach

## Get Size

```bash
gh api `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo
```

## Create big blob

```bash
gh api `
  --method POST `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo/git/blobs `
  --input ./blob-request-data.json
```

returns 

```json
{
  "sha": "ef586905f06d8b773ac61e7993af31567a5c9470",
  "url": "https://api.github.com/repos/lukasbach/test-repo/git/blobs/ef586905f06d8b773ac61e7993af31567a5c9470"
}
```

## Create a tree

```bash
gh api `
  --method POST `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo/git/trees `
  --input ./create-tree-request-data.json
```

returns

```json
{
  "sha": "66c6842dbd53f44a71c463aa7f64680a9d14b1ba",
  "url": "https://api.github.com/repos/lukasbach/test-repo/git/trees/66c6842dbd53f44a71c463aa7f64680a9d14b1ba",
  "tree": [
    {
      "path": "dummy.txt",
      "mode": "100644",
      "type": "blob",
      "sha": "ef586905f06d8b773ac61e7993af31567a5c9470",
      "size": 1555306,
      "url": "https://api.github.com/repos/lukasbach/test-repo/git/blobs/ef586905f06d8b773ac61e7993af31567a5c9470"
    }
  ],
  "truncated": false
}
```

## Create a commit

```bash
gh api `
  --method POST `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo/git/commits `
  --input ./create-commit-request-data.json
```

returns

```json
{
  "sha": "d0aa567bf512ddc9ce91b42d8722a9399c35dd03",
  "node_id": "C_kwDOIkHn29oAKGQwYWE1NjdiZjUxMmRkYzljZTkxYjQyZDg3MjJhOTM5OWMzNWRkMDM",
  "url": "https://api.github.com/repos/lukasbach/test-repo/git/commits/d0aa567bf512ddc9ce91b42d8722a9399c35dd03",
  "html_url": "https://github.com/lukasbach/test-repo/commit/d0aa567bf512ddc9ce91b42d8722a9399c35dd03",
  "author": {
    "name": "Lukas Bach",
    "email": "contact@lukasbach.com",
    "date": "2022-12-06T01:48:25Z"
  },
  "committer": {
    "name": "Lukas Bach",
    "email": "contact@lukasbach.com",
    "date": "2022-12-06T01:48:25Z"
  },
  "tree": {
    "sha": "66c6842dbd53f44a71c463aa7f64680a9d14b1ba",
    "url": "https://api.github.com/repos/lukasbach/test-repo/git/trees/66c6842dbd53f44a71c463aa7f64680a9d14b1ba"
  },
  "message": "my commit message",
  "parents": [
    {
      "sha": "6cd2a1d4e4f1259baeb3d744e9b37c0c7ba1b273",
      "url": "https://api.github.com/repos/lukasbach/test-repo/git/commits/6cd2a1d4e4f1259baeb3d744e9b37c0c7ba1b273",
      "html_url": "https://github.com/lukasbach/test-repo/commit/6cd2a1d4e4f1259baeb3d744e9b37c0c7ba1b273"
    }
  ],
  "verification": {
    "verified": false,
    "reason": "unsigned",
    "signature": null,
    "payload": null
  }
}
```

## Check the current main branch ref

```
gh api `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo/git/matching-refs/heads/main
```

returns

```json
[
  {
    "ref": "refs/heads/main",
    "node_id": "REF_kwDOIkHn269yZWZzL2hlYWRzL21haW4",
    "url": "https://api.github.com/repos/lukasbach/test-repo/git/refs/heads/main",
    "object": {
      "sha": "6cd2a1d4e4f1259baeb3d744e9b37c0c7ba1b273",
      "type": "commit",
      "url": "https://api.github.com/repos/lukasbach/test-repo/git/commits/6cd2a1d4e4f1259baeb3d744e9b37c0c7ba1b273"
    }
  }
]
```

## Update the main branch ref

```bash
gh api `
  --method PATCH `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo/git/refs/heads/main `
  -f sha='d0aa567bf512ddc9ce91b42d8722a9399c35dd03' `
 -F force=true 
```

returns

```json
{
  "ref": "refs/heads/main",
  "node_id": "REF_kwDOIkHn269yZWZzL2hlYWRzL21haW4",
  "url": "https://api.github.com/repos/lukasbach/test-repo/git/refs/heads/main",
  "object": {
    "sha": "d0aa567bf512ddc9ce91b42d8722a9399c35dd03",
    "type": "commit",
    "url": "https://api.github.com/repos/lukasbach/test-repo/git/commits/d0aa567bf512ddc9ce91b42d8722a9399c35dd03"
  }
}
```

# Overwrite existing previous commit

## Create new blob and a tree

This time in one. This is not referencing the previous tree, and creates a new one from scratch.

```bash
gh api `
  --method POST `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo/git/trees `
  --input ./create-tree-2-request-data.json
```

returns

```json
{
  "sha": "574c51a786971b89f2f8d63327d545577c950d78",
  "url": "https://api.github.com/repos/lukasbach/test-repo/git/trees/574c51a786971b89f2f8d63327d545577c950d78",
  "tree": [
    {
      "path": "dummy.txt",
      "mode": "100644",
      "type": "blob",
      "sha": "a3a3eedb481488d29ede333fc6c8d4422e489747",
      "size": 7,
      "url": "https://api.github.com/repos/lukasbach/test-repo/git/blobs/a3a3eedb481488d29ede333fc6c8d4422e489747"
    }
  ],
  "truncated": false
}
```

## Create a new commit

References previous tree, and original head commit, so other commit will be overwritten.

```bash
gh api `
  --method POST `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo/git/commits `
  --input ./create-commit-2-request-data.json
```

returns

```json
{
  "sha": "5779399ad7449c502eec5e145bcc9706798aa6e6",
  "node_id": "C_kwDOIkHn29oAKDU3NzkzOTlhZDc0NDljNTAyZWVjNWUxNDViY2M5NzA2Nzk4YWE2ZTY",
  "url": "https://api.github.com/repos/lukasbach/test-repo/git/commits/5779399ad7449c502eec5e145bcc9706798aa6e6",
  "html_url": "https://github.com/lukasbach/test-repo/commit/5779399ad7449c502eec5e145bcc9706798aa6e6",
  "author": {
    "name": "Lukas Bach",
    "email": "contact@lukasbach.com",
    "date": "2022-12-06T17:14:06Z"
  },
  "committer": {
    "name": "Lukas Bach",
    "email": "contact@lukasbach.com",
    "date": "2022-12-06T17:14:06Z"
  },
  "tree": {
    "sha": "574c51a786971b89f2f8d63327d545577c950d78",
    "url": "https://api.github.com/repos/lukasbach/test-repo/git/trees/574c51a786971b89f2f8d63327d545577c950d78"
  },
  "message": "overwritten commit",
  "parents": [
    {
      "sha": "6cd2a1d4e4f1259baeb3d744e9b37c0c7ba1b273",
      "url": "https://api.github.com/repos/lukasbach/test-repo/git/commits/6cd2a1d4e4f1259baeb3d744e9b37c0c7ba1b273",
      "html_url": "https://github.com/lukasbach/test-repo/commit/6cd2a1d4e4f1259baeb3d744e9b37c0c7ba1b273"
    }
  ],
  "verification": {
    "verified": false,
    "reason": "unsigned",
    "signature": null,
    "payload": null
  }
}
```

## Update the main branch ref

```bash
gh api `
  --method PATCH `
  -H "Accept: application/vnd.github+json" `
  /repos/lukasbach/test-repo/git/refs/heads/main `
  -f sha='5779399ad7449c502eec5e145bcc9706798aa6e6' `
 -F force=true 
```

returns

```json
{
  "ref": "refs/heads/main",
  "node_id": "REF_kwDOIkHn269yZWZzL2hlYWRzL21haW4",
  "url": "https://api.github.com/repos/lukasbach/test-repo/git/refs/heads/main",
  "object": {
    "sha": "5779399ad7449c502eec5e145bcc9706798aa6e6",
    "type": "commit",
    "url": "https://api.github.com/repos/lukasbach/test-repo/git/commits/5779399ad7449c502eec5e145bcc9706798aa6e6"
  }
}
```


## Locally removing unreferenced blobs

```bash
git -c gc.reflogExpire=0 -c gc.reflogExpireUnreachable=0 -c gc.rerereresolved=0 `
    -c gc.rerereunresolved=0 -c gc.pruneExpire=now gc
```