# Security Policy

This is a personal learning/reference infrastructure project (CloudFormation + AWS), not intended for production use. There are no supported version branches — only `main` is maintained.

## Reporting a Vulnerability

If you find a security issue — an exposed secret, a misconfigured resource, or a routing/firewall rule that undermines the inspection design — please report it privately using [GitHub's private vulnerability reporting](../../security/advisories/new) instead of opening a public issue.

## Scope

- This repository's CloudFormation templates and the AWS resources they define
- Accidentally committed credentials or parameter files with real values

Out of scope: vulnerabilities in AWS services (Gateway Load Balancer, Transit Gateway) themselves — please report those to AWS.
