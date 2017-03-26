# Getting Started

If you have followed the instructions in the mail we sent out before the workshop, you're already set. The only thing left to do is to join the Slack channel for the workshop, and start with the first lab session!

If you haven't completed the setup yet, please follow all of the steps below.

## The Slack channel

Throughout the workshop, we will use the Slack channel to send you additional information, updates and URLs. To join the channel, go to TODO

If you do not have a Slack account, you can simply create one during the joining process. No need to go through a complicated setup procedure.

## Workshop setup

During the workshop, you're going to work on a local Ember application. We are going to implement OAuth flows with a couple of providers, and inspect the security properties of these flows.

You are free to use your own personal development environment. We only need one specific security tool, which requires a few steps to set up. Instructions are included below.


### Access to the repo

If you haven't told us what your Github user name is, please do that now on the Slack channel, so we can invite you to the repo.

Once the invitation arrives, accept it and open the repository in your browser.


### Seting up the application and tools

Follow the README file of the repository, which tells you how to setup the applicationd and the tools. In a nutshell, you should perform the following steps:

- Clone the repository and install dependencies
- Launch the application (pay attention to the proxy options in the command)
- Create a user in the backend, using an email address you use on Github, Google or Facebook
- Setup Burp (a security tool) and configure Firefox to send traffic through Burp

### Create accounts with the OAuth providers (if necessary)

Make sure you have an account with the OAuth 2.0 providers we will be using. Concretely, you should have:

- A Google account
- Either a Github or a Facebook account

Don't worry about personal information in the account. The application we use will only request your email address. It does not look for any other information, and does not perform any actions in your name.