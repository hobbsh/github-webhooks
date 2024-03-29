# github-webhooks
Application that accepts github webhooks on repo create and sets branch protection  

## References
- https://groob.io/tutorial/go-github-webhook/
- https://github.com/krishbhanushali/go-rest-unit-testing
- A lot of the code in `go-github`, including tests: https://github.com/google/go-github/tree/master/test/integration
- StackOverflow

## Overview

I decided to write this application in Go because I have very little experience with the language and wanted to challenge myself. I have done something similar to this (processing GitHub webhooks) in Python so I figured I should branch out a bit more. This application can be run with docker, deployed to Kubernetes, or just run with `go run` locally. I've setup the webhook endpoint currently to just use ngrok. 

The `Repositories.GetBranch()` call intermittently returns a 404 which appears to be a result of the repo not being fully created before the webhook is processed. Initially I added a `time.Sleep()` but the `backoff` libarary handles the logic much better. See `getBranch()` in `main.go`.

## Running with docker
```
docker build -t github-webhook .
docker run -it --rm -e GITHUB_WEBHOOK_SECRET=$GITHUB_WEBHOOK_SECRET -e GITHUB_ACCESS_TOKEN=$GITHUB_ACCESS_TOKEN -p 8080:8080 github-webhook
```

Install and run ngrok as noted below. You will still need to Setup a GitHub Access Token and the webhook itself as noted below as well.

## Running the application locally

To setup your environment:

#### Install Go (on OSX)
- `curl -o golang.pkg https://dl.google.com/go/go1.13.3.darwin-amd64.pkg`
- `sudo open golang.pkg`

Add the necessary environment variables for Go:
```
echo "export GOROOT=/usr/local/go" >> ~/.bash_profile
echo "export GOPATH=$HOME/go" >> ~/.bash_profile 
echo "export PATH=$GOPATH/bin:$GOROOT/bin:$PATH" >> ~/.bash_profile
```

Install go dependencies:
- In the root of this repo, run: `go get -d ./...`

#### Install ngrok
- `brew cask install ngrok` (if you don't have brew, click [here](https://brew.sh/).)

#### Setup your GitHub Access Token
- Make sure you are an owner in your Organization.
- As your personal account, navigate to https://github.com/settings/tokens
- Create a new token with full control of repositories (`repo:*`) and `delete_repo` permissions
- Save it somewhere safe. You will also need to export it locally. On the cli, run `export GITHUB_ACCESS_TOKEN=<THE TOKEN>`

#### Running the application
- Start ngrok on port 8080: `ngrok http 8080`
- Make sure you are an owner in your Organization. Create a new Personal Access Token with 
- In GitHub, navigate to the Organization Webhooks settings page (https://github.com/organizations/<ORG NAME>/settings/hooks)
- Click Add Webhook
- Copy the HTTPS forwaring URL that ngrok displays and paste it in to the payload url with the correct path (`/webhook`) added to the end. It should look something like: `https://dad9f301.ngrok.io/webhook`.
- Type in anything for the webhook secret but make sure you can remember it because it needs to be exported as an environment variable
- After this, export the secret as an environment variable in your terminal session: `export GITHUB_WEBHOOK_SECRET=<THE SECRET>` 
- In the root of this repo, run: `go run main.go` in the same terminal session that you exported your environment variable (GITHUB_ACCESS_TOKEN, GITHUB_WEBHOOK_SECRET)
- Under "Which events to trigger" back in the Organization Webhooks page, select only `Repositories` events and save the webhook.
- The initial ping event should return 204

## Testing

There is a single end-to-end integration test included with this application that will test the entire process from start to finish, including creating/deleting a `testing` repo to perform the operations on. To run the test, be in the root of this repo and run:

`go test -v .`

You can navigate to the repo to validate that the branch protection has been enabled and that an issue explaining that has been created. The organization name is hardcoded into the test. The test will leave the testing repo in place for manual validation and destroys it at the beginning of each run.
