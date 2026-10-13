# Slack configuration recipes

[Multiple Slack workspaces in the Codex desktop app](multi-slackspace.md) explains prerequisites, approved OAuth clients, separate server aliases, workspace authorization, configuration examples, verification, troubleshooting, and removal.

Use the approved integration's supplied configuration, create one named connection per workspace, and sign in separately with the user's own Slack account. A client ID may already be supplied; manual ID entry is only an optional setup path. Placeholder IDs illustrate the configuration and cannot authenticate. Repository access does not provide Slack access. The recipe can be shared independently of the rest of the repository.

Examples use generic workspace labels and contain no operational credentials or workspace identifiers. Keep authentication files, configuration backups, and private Slack content outside the repository.
