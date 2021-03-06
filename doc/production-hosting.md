# Production hosting

[![operations issues](https://img.shields.io/github/issues/badges/shields/operations.svg?label=open%20operations%20issues)][operations issues]

[#ops chat room][ops discord]

[operations issues]: https://github.com/badges/shields/issues?q=is%3Aissue+is%3Aopen+label%3Aoperations
[ops discord]: https://discordapp.com/channels/308323056592486420/480747695879749633

| Component                     | Subcomponent                    | People with access                                                                         |
| ----------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------ |
| Badge servers                 | Account owner                   | @espadrine                                                                                 |
| Badge servers                 | ssh, logs                       | @espadrine                                                                                 |
| Badge servers                 | Deployment                      | @espadrine, @paulmelnikow                                                                  |
| Badge servers                 | Admin endpoints                 | @espadrine, @paulmelnikow                                                                  |
| Compose.io Redis              | Account owner                   | @paulmelnikow                                                                              |
| Compose.io Redis              | Account access                  | @paulmelnikow                                                                              |
| Compose.io Redis              | Database connection credentials | @espadrine, @paulmelnikow                                                                  |
| Zeit Now                      | Team owner                      | @paulmelnikow                                                                              |
| Zeit Now                      | Team members                    | @paulmelnikow, @chris48s, @calebcartwright, @platan                                        |
| Raster server                 | Full access as team members     | @paulmelnikow, @chris48s, @calebcartwright, @platan                                        |
| shields-server.com redirector | Full access as team members     | @paulmelnikow, @chris48s, @calebcartwright, @platan                                        |
| Cloudflare                    | Account owner                   | @espadrine                                                                                 |
| Cloudflare                    | Admin access                    | @espadrine, @paulmelnikow                                                                  |
| GitHub                        | OAuth app                       | @espadrine ([could be transferred to the badges org][oauth transfer])                      |
| Twitch                        | OAuth app                       | @PyvesB                                                                                    |
| OpenStreetMap (for Wheelmap)  | Account owner                   | @paulmelnikow                                                                              |
| DNS                           | Account owner                   | @olivierlacan                                                                              |
| DNS                           | Read-only account access        | @espadrine, @paulmelnikow, @chris48s                                                       |
| Sentry                        | Error reports                   | @espadrine, @paulmelnikow                                                                  |
| Frontend                      | Deployment                      | Technically anyone with push access but in practice must be deployed with the badge server |
| Metrics server                | Owner                           | @platan                                                                                    |
| UptimeRobot                   | Account owner                   | @paulmelnikow                                                                              |
| More metrics                  | Owner                           | @RedSparr0w                                                                                |
| Netlify (documentation site)  | Owner                           | @chris48s                                                                                  |

There are [too many bottlenecks][issue 2577]!

[issue 2577]: https://github.com/badges/shields/issues/2577

## Attached state

Shields has mercifully little persistent state:

1.  The GitHub tokens we collect are saved on each server in a cloud Redis database.
    They can also be fetched from the [GitHub auth admin endpoint][] for debugging.
2.  The server keeps a few caches in memory. These are neither persisted nor
    inspectable.
    - The [request cache][]
    - The [regular-update cache][]

[github auth admin endpoint]: https://github.com/badges/shields/blob/master/services/github/auth/admin.js
[request cache]: https://github.com/badges/shields/blob/master/core/base-service/legacy-request-handler.js#L29-L30
[regular-update cache]: https://github.com/badges/shields/blob/master/core/legacy/regular-update.js
[oauth transfer]: https://developer.github.com/apps/managing-oauth-apps/transferring-ownership-of-an-oauth-app/

## Configuration

To bootstrap the configuration process,
[the script that starts the server][start-shields.sh] sets a single
environment variable:

```
NODE_CONFIG_ENV=shields-io-production
```

With that variable set, the server ([using `config`][config]) reads these
files:

- [`local-shields-io-production.yml`][local-shields-io-production.yml].
  This file contains secrets which are checked in with a deploy commit.
- [`shields-io-production.yml`][shields-io-production.yml]. This file
  contains non-secrets which are checked in to the main repo.
- [`default.yml`][default.yml]. This file contains defaults.

[start-shields.sh]: https://github.com/badges/ServerScript/blob/master/start-shields.sh#L7
[config]: https://github.com/lorenwest/node-config/wiki/Configuration-Files
[local-shields-io-production.yml]: ../config/local-shields-io-production.template.yml
[shields-io-production.yml]: ../config/shields-io-production.yml
[default.yml]: ../config/default.yml

The project ships with `dotenv`, however there is no `.env` in production.

## Badge CDN

Sitting in front of the three servers is a Cloudflare Free account which
provides several services:

- Global CDN, caching, and SSL gateway for `img.shields.io`
- Analytics through the Cloudflare dashboard
- DNS hosting for `shields.io`

Cloudflare is configured to respect the servers' cache headers.

## Frontend

The frontend is served by [GitHub Pages][] via the [gh-pages branch][gh-pages]. SSL is enforced.

`shields.io` resolves to the GitHub Pages hosts. It is not proxied through
Cloudflare.

Technically any maintainer can push to `gh-pages`, but in practice the frontend must be deployed
with the badge server via the deployment process described below.

[github pages]: https://pages.github.com/
[gh-pages]: https://github.com/badges/shields/tree/gh-pages

## Raster server

The raster server `raster.shields.io` (a.k.a. the rasterizing proxy) is
hosted on [Zeit Now][]. It's managed in the
[svg-to-image-proxy repo][svg-to-image-proxy].

[zeit now]: https://zeit.co/now
[svg-to-image-proxy]: https://github.com/badges/svg-to-image-proxy

## Deployment

The deployment is done in two stages: the badge server (heroku) and the front-end (gh-pages).

### Heroku

After merging a commit to master, heroku should create a staging deploy. Check this has deployed correctly in the `shields-staging` pipeline and review http://shields-staging.herokuapp.com/

If we're happy with it, "promote to production". This will deploy what's on staging to the `shields-production-eu` and `shields-production-us` pieplines.

### Frontend

To deploy the front-end to GH pages, use a clean clone of the shields repo.

```sh
$ git pull  # update the working copy
$ npm ci  # install dependencies (devDependencies are needed to build the frontend)
$ make deploy-gh-pages  # build the frontend and push it to the gh-pages branch
```

No secrets are required to build or deploy the frontend.

## DNS

DNS is registered with [DNSimple][].

[dnsimple]: https://dnsimple.com/

## Logs

Logs can be retrieved [from heroku](https://devcenter.heroku.com/articles/logging#log-retrieval).

## Error reporting

[Error reporting][sentry] is one of the most useful tools we have for monitoring
the server. It's generously donated by [Sentry][sentry home]. We bundle
[`raven`][raven] into the application, and the Sentry DSN is configured via
`local-shields-io-production.yml` (see [documentation][sentry configuration]).

[sentry]: https://sentry.io/shields/
[raven]: https://www.npmjs.com/package/raven
[sentry home]: https://sentry.io/shields/
[sentry configuration]: https://github.com/badges/shields/blob/master/doc/self-hosting.md#sentry

## Monitoring

Overall server performance and requests by service are monitored using
[Prometheus and Grafana][metrics].

Request performance is monitored in two places:

- [Status][] (using [UptimeRobot][])
- [Server metrics][] using Prometheus and Grafana
- [@RedSparr0w's monitor][monitor] which posts [notifications][] to a private
  [#monitor chat room][monitor discord]

[metrics]: https://metrics.shields.io/
[status]: https://status.shields.io/
[server metrics]: https://metrics.shields.io/
[uptimerobot]: https://uptimerobot.com/
[monitor]: https://shields.redsparr0w.com/1568/
[notifications]: http://shields.redsparr0w.com/discord_notification
[monitor discord]: https://discordapp.com/channels/308323056592486420/470700909182320646

## Legacy servers

There are three legacy servers on OVH VPS’s which are currently used for proxying.

| Cname                       | Hostname             | Type | IP             | Location           |
| --------------------------- | -------------------- | ---- | -------------- | ------------------ |
| [s0.servers.shields.io][s0] | vps71670.vps.ovh.ca  | VPS  | 192.99.59.72   | Quebec, Canada     |
| [s1.servers.shields.io][s1] | vps244529.ovh.net    | VPS  | 51.254.114.150 | Gravelines, France |
| [s2.servers.shields.io][s2] | vps117870.vps.ovh.ca | VPS  | 149.56.96.133  | Quebec, Canada     |

[s0]: https://s0.servers.shields.io/index.html
[s1]: https://s1.servers.shields.io/index.html
[s2]: https://s2.servers.shields.io/index.html

The only way to inspect the commit on the server is with `git ls-remote`.
