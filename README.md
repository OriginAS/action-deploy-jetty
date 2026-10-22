# action-deploy-jetty

Deploys a `.war` to a Jetty instance on an EC2 server, over SSH, from a GitHub-hosted runner.

The action opens SSH in the server's security group for the runner's IP, uploads the files and runs
`~/deploy-jetty` (and `~/deploy-file` for config) on the server, then revokes the IP again. These are
the same scripts the self-hosted runner used, so nothing new needs to be rolled out to the servers.

`~/deploy-jetty` creates the Jetty instance from `/var/jetty/template` if it does not exist, waits for
the webapp to reach `STARTED`, and puts the previous `.war` back if it does not.

## Usage

The caller builds the `.war`, so secrets can be injected and Maven profiles chosen as before:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: 11
          distribution: 'temurin'
      - name: Inject secrets in application.properties
        run: sed -i 's/$DB_PASSWORD/${{ secrets.DB_PASSWORD }}/g' ./src/main/resources/staging/application.properties
      - run: mvn -B -Dmaven.test.skip -Pstaging clean package

      - uses: OriginAS/action-deploy-jetty@main
        with:
          aws-access-key-id: ${{ secrets.ACTIONS_AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.ACTIONS_AWS_ACCESS_SECRET }}
          aws-security-group-id: sg-08c47dbcf42a2a6c6
          application-name: 'myapp'
          application-file: ./target/myapp-*.war
          jetty-instance: myapp
          server-host: origin-staging-01.aws.origin.no
          server-ssh-key: ${{ secrets.ACTIONS_SSH_PRIVATE_KEY }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `aws-access-key-id` | yes | | AWS key allowed to change the security group |
| `aws-secret-access-key` | yes | | |
| `aws-security-group-id` | yes | | Security group to open SSH on for the runner |
| `aws-region` | no | `eu-north-1` | Region the security group lives in |
| `application-name` | yes | | Context: the `.war` is installed as `<application-name>.war` |
| `application-file` | yes | | The `.war` to deploy. A glob must match exactly one file |
| `jetty-instance` | yes | | Jetty instance under `/var/jetty/` |
| `application-properties` | no | | Installed as `/var/jetty/<instance>/resources/application.properties`, before the `.war` |
| `application-vhost` | no | | Installed as `/etc/httpd/vhosts.d/<application-name>.conf`, then httpd is reloaded |
| `server-host` | yes | | Host name of the server |
| `server-ssh-key` | yes | | SSH private key for `server-user` |
| `server-user` | no | `actions` | User to connect as |

When a server script fails, the action prints the tail of `/opt/actions/deploy.log`, since that is
where the scripts write their output.
