# docker-gitlab

Bootstraps a fresh EC2 instance with EFS and GitLab with CI

## Usage

*Note*: This play assumes you're deploying to AWS. YMMV on other platforms.

At its most basic: `ansible-playbook local.yml`

When implementing plays like this, it is best practice to bake an AMI which runs the playbook on startup.  [inhumantsar.bootstrap](https://github.com/inhumantsar/ansible-role-bootstrap) will set up a systemd service which performs an `ansible-pull` to clone/pull the playbook repo prior to running it at startup. This role can be [implemented with Packer](https://www.packer.io/docs/provisioners/ansible.html) during the image bake process.

## Configuration

* The EFS mountpoint must already exist on a subnet GitLab can access.
* Instance must already have read/write permissions on the EFS mountpoint and the S3 bucket listed in `efs_url` and `gitlab_backups_bucket`.
  * If the S3 bucket doesn't already exist, then create permissions must also be included.
* The domain does *not* have to be a real resolvable domain, even in production, but it will make life easier if it is.
* `ci_runner_registration_token` can be left blank for better data security, but the CI runner would have to be registered manually after deploy. A better solution would be to reach out to a secrets management system or to use Ansible Vault features to encrypt it.
* Other deployment options are available. See the [inhumantsar.docker-compose-gitlab role](https://github.com/inhumantsar/ansible-docker-compose-gitlab/blob/master/defaults/main.yml) for more information.
* GitLab configuration files are *huge* and aren't always clear. Do not rely solely on the sample gitlab.rb file included in this repo. [RTFM](https://docs.gitlab.com/omnibus/settings/configuration.html)

### Off-Peak Periods
```bash
  --machine-off-peak-periods "* * 0-6,18-23 * * mon-fri *" \
  --machine-off-peak-periods "* * * * * sat,sun *" \
  --machine-off-peak-timezone "America/Vancouver" \
  --machine-off-peak-idle-count 0 \
  --machine-off-peak-idle-time 5 \
```

These three options for the CI runner registration command set off-peak periods. In layman's terms, it will not keep any CI runners alive from 6pm to 6am Pacific Time unless someone triggers a job during that time. If a job is triggered and a runner launched, it will only keep the instance alive for 5 minutes of idle time.

## Known Issues

* `executor = "docker+machine"` in the config file results in _invalid executor_ errors, using an environment variable or a command line option instead fixes it. [more info](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/issues/2198#note_24973490)
* When registering a CI runner, it's not wise to mix env vars and config file options because the runner is terrible at recognizing and using existing configs. In my experience, it replaces the named config with a new generic one and moves the old configs to an unnamed config. This results in a broken runner. I found that the clearest, most extensible way to set these configs was to pass them as command line options to the `gitlab-runner register` command.
* Gitlab's current `alpine` Docker image tag uses an old version of Docker Machine which can no longer deploy new slaves correctly. Machine always installs the latest version of Docker, but Machine versions <0.12.2 cannot work with Docker versions > 17ish. The current `alpine-bleeding` image is due to be promoted soon. If you're reading this in 2018, you may not need `bleeding` anymore.
