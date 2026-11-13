# Linux Server Monitoring with Grafana Cloud

A Linux host-monitoring project built on **Linux Mint**, **Grafana Alloy**, **Prometheus Node Exporter**, and **Grafana Cloud**. The project configures host metrics and system-log collection, forwards telemetry to Grafana Cloud, and uses dashboards and alert rules to monitor system health.

> **Project status:** The core monitoring project is complete. Grafana Alloy is installed as a systemd service and configured to collect and forward host telemetry. Grafana Cloud dashboards and a CPU-utilisation alert are part of the setup. Use the verification steps in this README to confirm live data ingestion and notification delivery in the target Grafana Cloud stack.

---

## 1. Architecture

```text

                    Linux Mint host

                         |

          +--------------+---------------+

          |                              |

    Node Exporter                   Host log files

    (host metrics)                  (system logs)

          |                              |

          +--------------+---------------+

                         |

                    Grafana Alloy

                 (alloy.service / systemd)

                         |

             +-----------+-----------+

             |                       |

       Prometheus remote_write    Loki write

            (metrics)               (logs)

             |                       |

             v                       v

       Grafana Cloud             Grafana Cloud

       Prometheus/Mimir                Loki

             |                       |

             +-----------+-----------+

                         |

                 Grafana Cloud UI

                         |

             +-----------+-----------+

             |                       |

        Dashboards               Alert rules

      CPU/system,              CPU threshold

    filesystem/disks           and contact point

```

### Components

- **Linux Mint:** The monitored host.

- **Prometheus Node Exporter:** Exposes operating-system metrics, commonly on port `9100`.

- **Grafana Alloy:** Collects and forwards metrics and logs. It runs as `alloy.service`.

- **Grafana Cloud Metrics:** Receives Prometheus-compatible metrics through the remote-write endpoint.

- **Grafana Cloud Logs (Loki):** Receives log data through the Loki push endpoint.

- **Grafana dashboards:** Visualise CPU, memory, network, uptime, disks and filesystem information.

- **Grafana alerting:** Evaluates queries and can send notifications through configured contact points.

## 2. Prerequisites

- A Linux Mint machine with internet access.

- A Grafana Cloud account and a Grafana Cloud stack.

- `sudo` access on the Linux host.

- Grafana Cloud write credentials with the required permissions:

  - `metrics:write` for metrics ingestion.

  - `logs:write` for log ingestion.

- Grafana Alloy installed using the Grafana Cloud **Monitor your OS** integration instructions.

- Node Exporter installed and running if it is part of the generated integration configuration.

Do not commit API tokens, passwords, private keys, or other credentials to Git.

## 3. Setup instructions

### 3.1 Install Alloy

In Grafana Cloud, open the OS monitoring integration (often shown as **Monitor your OS**) and follow the installation instructions for the appropriate Linux distribution and architecture.

The installation creates an Alloy systemd service and a configuration file. In the environment used for this project, the relevant paths are:

```text

/etc/alloy/config.alloy

/etc/systemd/system/alloy.service.d/env.conf

/etc/systemd/system/alloy.service.d/alloy.env

```

The systemd drop-in file `env.conf` references `alloy.env` as an environment file. The credentials are stored in `alloy.env`, not directly in the Alloy configuration.

These paths describe this installation and may differ on another Linux distribution or installation method.

### 3.2 Configure credentials

Edit the environment file:

```bash

sudo nano /etc/systemd/system/alloy.service.d/alloy.env

```

The installation uses environment variables including:

```text

GCLOUD_RW_API_KEY=<Grafana-Cloud-write-token>

GCLOUD_FM_COLLECTOR_ID=<collector-id>

```

The Alloy configuration references the token through:

```alloy

password = sys.env("GCLOUD_RW_API_KEY")

```

In this setup, the same variable is referenced by both the Prometheus remote-write and Loki write components. Confirm that the credential is valid for the configured destinations and has the required write permissions. If separate credentials are required, configure separate environment variables and update the corresponding component references.

Secure the file and restart Alloy after credential changes:

```bash

sudo chmod 600 /etc/systemd/system/alloy.service.d/alloy.env

sudo systemctl daemon-reload

sudo systemctl restart alloy

```

Never paste tokens into issue reports, screenshots, terminal transcripts, or Git commits.

### 3.3 Verify the service

```bash

sudo systemctl status alloy.service --no-pager -l

```

Check the environment-file configuration:

```bash

sudo systemctl show alloy --property=EnvironmentFiles

```

The setup used for this project loads the following environment files:

```text

/etc/default/alloy

/etc/systemd/system/alloy.service.d/alloy.env

```

Check that the running process received the expected variable names without displaying their values:

```bash

pid=$(systemctl show -p MainPID --value alloy)

sudo sh -c "tr '\0' '\n' < /proc/$pid/environ | grep '^GCLOUD_' | sed 's/=.*/=[REDACTED]/'"

```

Expected variable names include `GCLOUD_RW_API_KEY` and `GCLOUD_FM_COLLECTOR_ID`.

### 3.4 Inspect the Alloy configuration

```bash

sudo less /etc/alloy/config.alloy

```

The generated configuration should contain components for Prometheus remote write and Loki write. It should reference the Grafana Cloud endpoints and obtain credentials from environment variables.

Avoid publishing the full configuration without checking it for embedded secrets.

## 4. Dashboard metrics

The project uses dashboards to monitor host performance and storage.

### CPU and system

Use this as the main host overview. Depending on the dashboard version and available metrics, it can show:

- CPU utilisation and CPU load.

- Memory usage and availability.

- Network receive and transmit traffic.

- System uptime.

- General host-level performance information.

### Filesystem and disks

Use this dashboard for storage monitoring, including:

- Filesystem capacity.

- Used and available disk space.

- Filesystem utilisation.

- Disk-related activity metrics, where collected.

The exact panels and metric availability depend on the dashboard version and the exporters and collectors enabled in Alloy.

## 5. Alert configuration

The initial CPU alert is intended to warn when CPU utilisation exceeds **80%**.

### Query and expressions

1. Create a query (`A`) that returns CPU utilisation as a percentage on a 0–100 scale.

2. Add a **Reduce** expression:

   - Input: `A`

   - Function: `Last`

   - Mode: `Strict`

3. Add a **Threshold** expression:

   - Input: `B` (the Reduce expression)

   - Condition: `Is above`

   - Value: `80`

4. Set the threshold expression as the alert condition.

5. Preview the rule and save it.

The Reduce expression selects the latest query value. The Threshold expression evaluates whether that value is above 80.

> **Important:** Confirm that query `A` really returns a percentage. A raw CPU counter or a value on a 0–1 scale requires a different query or threshold.

### Notifications

Grafana contact points determine where alert notifications are sent. The email address must be permitted by the Grafana Cloud organisation's recipient settings. If Grafana rejects an address because it is not recognised as an organisation member, check the organisation's user list and email-recipient restrictions, or use another supported notification integration.

Do not assume that saving an alert rule proves notifications work. Test the contact point and verify that a notification is received before describing email delivery as operational.

## 6. Testing and validation

Use the following checks after installation or configuration changes.

### 6.1 Check Alloy service health

```bash

systemctl is-active alloy

sudo systemctl status alloy --no-pager -l

```

Expected service state:

```text

active

```

### 6.2 Review recent logs

```bash

sudo journalctl -u alloy.service -n 100 --no-pager

```

To filter for common failures:

```bash

sudo journalctl -u alloy.service --since today --no-pager \

  | grep -iE 'error|401|403|unauthorized|failed|retry|timeout'

```

A quiet journal does not necessarily mean that ingestion is working; verify data in Grafana Cloud as well.

### 6.3 Verify metrics in Grafana Cloud

Open **Explore** in the Grafana Cloud stack, select the Prometheus metrics data source, and query metrics such as:

```promql

up

```

```promql

node_cpu_seconds_total

```

```promql

node_memory_MemAvailable_bytes

```

```promql

node_network_receive_bytes_total

```

```promql

node_filesystem_size_bytes

```

Metric names can vary depending on the exporter and integration configuration. If a query returns no data, check the integration's actual scrape targets and metric names.

### 6.4 Verify logs

Open **Explore**, select the Loki data source, and query the labels available for the host. Inspect the label browser first, then build a query using labels that are actually present.

### 6.5 Verify alerting

- Use **Preview** to check the query and expressions.

- Confirm that the alert condition uses the Threshold expression.

- Use the contact point's **Test** action.

- Confirm that the destination receives the notification.

- Only then consider the notification workflow tested.

Do not deliberately overload a production machine just to test a CPU alert. Use Grafana's preview/test features or a controlled, brief test on a non-critical machine.

## 7. Troubleshooting

### `401 Unauthorized` or `403 Forbidden`

Check that the credential configured in the Alloy environment file is valid, has not expired or been revoked, and has the required write permissions for the relevant Grafana Cloud services. Confirm that Alloy is loading the intended environment file, then restart the service after making changes.

### DNS lookup errors

An error such as:

```text

lookup prometheus-prod-...grafana.net: server misbehaving

```

indicates a DNS resolution problem. Check internet connectivity and DNS resolution:

```bash

getent hosts prometheus-prod-43-prod-ap-south-1.grafana.net

getent hosts logs-prod-028.grafana.net

```

If name resolution fails, investigate the host's DNS configuration or network before changing Grafana credentials.

### Out-of-order or old sample timestamp

An error such as:

```text

err-mimir-sample-timestamp-too-old

```

means the metrics backend rejected samples whose timestamps are outside its accepted time window. Check the host clock and time synchronisation:

```bash

timedatectl status

```

Correct any clock or time-synchronisation issue. Avoid repeatedly replaying stale buffered samples.

### Permission denied reading `/var/log/boot.log`

If Alloy reports that it cannot read a log file, check the file permissions and the service's access. Do not make all system logs world-readable as a quick fix. Grant only the minimum required access using a supported service/group configuration.

### No data in dashboards

- Confirm Alloy is active.

- Check recent Alloy logs for remote-write or Loki errors.

- Confirm the correct Prometheus or Loki data source is selected.

- Check the time range in the dashboard.

- Confirm that the relevant exporter and scrape components are running.

- Verify that metrics are arriving in Explore before troubleshooting dashboard panels.

### Email contact point cannot be saved

If Grafana reports that the recipient email is not permitted or is not a member of the organisation, check the user list and the organisation's email-recipient restrictions. Use a permitted address or another supported notification integration. Test the contact point after saving.

## 8. Security and operational notes

- Keep tokens in protected environment files, not in source control.

- Use restrictive permissions on secret files (`chmod 600`).

- Rotate any token that has been exposed.

- Redact credentials from logs and screenshots before sharing them.

- Avoid changing generated Alloy endpoints or tenant identifiers unless verified against the Grafana Cloud stack's own instructions.

- Recheck service status and ingestion after every credential or configuration change.

- Review the repository before pushing to ensure that no environment files, credentials, or private configuration have been added.

## 9. Project completion checklist

Use this checklist to verify the project before considering a deployment complete:

- [ ] Alloy is installed and enabled as a systemd service.

- [ ] Alloy starts successfully after reboot.

- [ ] Credentials are stored securely and accepted by Grafana Cloud.

- [ ] Prometheus metrics are visible in Grafana Cloud Explore.

- [ ] Loki logs are visible in Grafana Cloud Explore.

- [ ] CPU and system dashboard displays host data.

- [ ] Filesystem and disks dashboard displays storage data.

- [ ] CPU alert evaluates the intended percentage query and threshold.

- [ ] Contact point saves successfully and a test notification is received, if email notifications are required.

- [ ] Setup and troubleshooting steps are documented without secrets.

## 10. Environment-specific reference

The following details describe the environment used while creating this project. Treat them as environment-specific, not universal defaults. Machine-specific identifiers and tenant details are intentionally omitted from this public-facing README.

| Item | Value |

|---|---|

| Host OS | Linux Mint |

| Alloy service | `alloy.service` |

| Alloy configuration | `/etc/alloy/config.alloy` |

| Environment file | `/etc/systemd/system/alloy.service.d/alloy.env` |

| Systemd drop-in | `/etc/systemd/system/alloy.service.d/env.conf` |

| Metrics endpoint region | Grafana Cloud `ap-south-1` |

| Scrape interval from setup | `60s` |

