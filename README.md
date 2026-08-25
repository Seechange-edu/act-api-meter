# act-api-meter

Deploy workflow for [api-meter](https://github.com/Seechange-edu/api-meter). This
repo holds no source — it exists so a Release created by the source repo can
trigger a build without giving the source repo the deploy credentials.

api-meter is a monorepo, so unlike the other Action Factory repos this one
builds **two images and runs three containers**:

| container | image | what it is |
| --- | --- | --- |
| admin | backend | management API; the operator UI's data |
| proxy | backend | what applications call — `/gw` forwarding and `/v1` ingest |
| web | frontend | nginx: the UI, with `/api` forwarded to admin |

admin and proxy are the same image with a different `SERVICE`, deployed apart so
a heavy report query cannot add latency to a path sitting in front of a paid API
call. All three share a user-defined docker network so nginx can reach admin by
name.

Migrations run before anything starts, and a failure stops the deploy with the
previous containers still up.

## Partial deploys

The source repo's workflows are split by path and put `DEPLOY_PARTS` in the
Release body — `backend`, `frontend`, or `all`. A frontend change therefore does
not restart the proxy, which sits in front of paid API calls and drops whatever
was in flight when it goes down.

The web container restarts whenever admin does, whichever part triggered the
deploy: nginx resolves its upstream once at start-up, so leaving it running
would leave it pointing at the address the previous admin container had.

Deploys share one concurrency group and queue rather than cancel. A commit
touching both halves fires two workflows and lands two Releases here at once,
and cancelling one mid-restart would leave the host in whatever state it had
reached.

## Setup

Settings → Secrets and variables → Actions:

- **Secrets:** `ACTION_TOKEN` (can check out `api-meter`), `REMOTE_SSH_KEY`
- **Variables:** `AWS_ROLE_ARN`, `AWS_REGION`, `REMOTE_USER`,
  `AWS_ECR_REPOSITORY` (`api-meter`)

Two ECR repositories must exist per environment — ECR does not create them on
push:

```
api-meter/dev/backend      api-meter/dev/frontend
api-meter/prod/backend     api-meter/prod/frontend
```

Ports, container names and memory limits live in
[Parse-Action-Tag](https://github.com/Seechange-edu/Parse-Action-Tag)'s
`REPOSITORY_ENV_MAP`, not here.
