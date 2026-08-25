<!--
SPDX-FileCopyrightText: 2020 - 2024 MDAD project contributors
SPDX-FileCopyrightText: 2020 - 2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 - 2025 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up ConvertX

This is an [Ansible](https://www.ansible.com/) role which installs [ConvertX](https://github.com/C4illin/ConvertX) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

ConvertX is a self-hosted online file converter which supports a lot of different formats for pictures, video, images, document files, etc.

See the project's [documentation](https://github.com/C4illin/ConvertX/blob/main/README.md) to learn what ConvertX does and why it might be useful to you.

## Adjusting the playbook configuration

To enable ConvertX with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# convertx                                                             #
#                                                                      #
########################################################################

convertx_enabled: true

########################################################################
#                                                                      #
# /convertx                                                            #
#                                                                      #
########################################################################
```

### Set the hostname

To enable ConvertX you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
convertx_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

### Set a random string for signing authentication tokens

You also need to set a random string for signing authentication tokens. To do so, add the following configuration to your `vars.yml` file. The value can be generated with `pwgen -s 64 1` or in another way.

```yaml
convertx_environment_variables_jwt_secret: YOUR_SECRET_KEY_HERE
```

### Create the first account promptly

To use ConvertX you need to create an account and log in to it on the browser.

>[!IMPORTANT]
> Until that first account exists, ConvertX serves an unauthenticated "Create your account" form at `/setup`, and accepts a `POST /register` regardless of `convertx_environment_variables_account_registration`. That is how the first account is meant to be created, but between the moment the service becomes reachable and the moment you register, anyone who finds the hostname can claim the instance instead of you. Once they do, the window closes against you as well: with account registration disabled you would have to remove the `mydb.sqlite` database from the role's data directory (`convertx_data_path`) on the server and start over.
>
> Create the account immediately after the first installation. If the hostname is public and you cannot do that right away, put the service behind HTTP Basic authentication for the meantime:
>
> ```yaml
> convertx_container_labels_traefik_middleware_basic_auth_enabled: true
> convertx_container_labels_traefik_middleware_basic_auth_users: YOUR_HTPASSWD_LINE_HERE
> ```

### Enable account registration (optional)

Account registration is disabled by default, which means that only the first account (see above) can be created through the web interface. To let anyone register an account, add the following configuration to your `vars.yml` file:

```yaml
convertx_environment_variables_account_registration: true
```

>[!WARNING]
> ConvertX accounts are not separated into administrators and regular users, and registration has no approval step or invitation mechanism. Enabling this on a publicly reachable hostname lets anyone run ffmpeg, LibreOffice, ImageMagick and the other bundled converters on your server with files of their choosing.
>
> Weigh that against what an account currently grants. ConvertX's LaTeX converter runs `latexmk` on uploaded documents without restricting what they may read, so an account holder can upload a `.tex` file that pulls in a file from the server and read it back out of the resulting PDF — including ConvertX's own database of accounts and password hashes. This was reported upstream as [GHSA-qwm5-vvqj-wrhc](https://github.com/C4illin/ConvertX/security/advisories/GHSA-qwm5-vvqj-wrhc) and is tracked as [issue #573](https://github.com/C4illin/ConvertX/issues/573); it is unfixed in the version this role installs. The container runs unprivileged with a read-only root filesystem, which limits the damage, but it does not prevent this.
>
> Treat every ConvertX account as trusted, and prefer creating accounts yourself over leaving registration open.

### Pass extra arguments to ffmpeg (optional)

ConvertX hands two separate sets of arguments to ffmpeg, and which one you need depends on where the argument belongs on an ffmpeg command line.

```yaml
# Input options - these go in front of `-i`
convertx_environment_variables_ffmpeg_args: -hwaccel vaapi

# Output options - these go after the input file
convertx_environment_variables_ffmpeg_output_args: -preset veryfast
```

Putting an output option such as `-preset` or `-crf` into `convertx_environment_variables_ffmpeg_args` makes ffmpeg reject the command, so every video conversion on the instance fails.

### Set a subpath (optional)

It is possible to serve the instance under a subpath by adding the following configuration to your `vars.yml` file.

```yaml
convertx_environment_variables_webroot: YOUR_SUBPATH_HERE
```

For example, setting this to `/convert` will have the website served on `https://example.com/convert/`.

>[!NOTE]
> The subpath cannot be specified with the `convertx_path_prefix` variable.

### Disabling authentication function

If the service is hosted locally or with an authentication service like [Tinyauth](https://tinyauth.app/), you can disable the authentication function of ConvertX in favor of it by adding the following configuration to your `vars.yml` file.

```yaml
convertx_environment_variables_allow_unauthenticated: true
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `convertx_environment_variables_additional_variables` variable

See [this section on the official documentation](https://github.com/C4illin/ConvertX/blob/main/README.md#environment-variables) for a complete list of ConvertX's config options that you could put in `convertx_environment_variables_additional_variables`.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, ConvertX becomes available at the specified hostname like `https://example.com`. To use it, open the URL on the browser and create an account.

Note that it is not available to restore the password if it is lost. In this case, you will need to uninstall the service and reinstall it to start it over.

## Troubleshooting

### Document conversions fail with "User installation could not be completed"

The container runs unprivileged with a read-only root filesystem, and LibreOffice — which handles the Office, OpenDocument and other document formats — refuses to start unless it can create a user profile somewhere writable. `convertx_environment_variables_xdg_config_home` points it at the container's `/tmp` for that, and it defaults to a working value; the failure above means it was overridden with a directory the container cannot write to, or one whose parent does not exist (LibreOffice creates only the last path component).

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu convertx` (or how you/your playbook named the service, e.g. `mash-convertx`).
