# mythos.darkspar.com

Source for the [Mythos](https://gitlab.com/darkspar/mythos) project site.

Live at <https://mythos.darkspar.com/>.

## Stack

[Hugo](https://gohugo.io/) (extended), pinned to `0.161.1`.

## Local development

```sh
hugo server
```

## Deployment

Pushes to `main` on the [gitlab.com remote](https://gitlab.com/darkspar/mythos.darkspar.com)
trigger `.gitlab-ci.yml`, which builds with `hugomods/hugo:std-0.161.1` and
publishes to GitLab Pages. The custom domain `mythos.darkspar.com` is served
with a Let's Encrypt certificate and force-HTTPS enabled.

Pushes to other remotes (self-hosted GitLab, GitHub mirror) do not publish —
the pipeline is gated to `$CI_SERVER_HOST == "gitlab.com"`.
