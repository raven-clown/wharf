# Security

Please don't report a security problem in a public issue.

Use GitHub's private vulnerability reporting instead: open the Security tab of this repository and choose "Report a vulnerability". You'll get a first answer within three days.

A registry holds credentials and decides who can pull what, so these count as security problems: a way to read or write a repository without the right permission, a token or password that leaks into logs or responses, a way past a signature or vulnerability gate, and anything in the Harbor importer that exposes secrets from the source database.

Fixes go to the latest release. Before 1.0, only the latest minor version gets security fixes.
