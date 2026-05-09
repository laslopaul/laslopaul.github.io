---
title: "Managing self-signed TLS for Docker Compose with step-ca"
date: 2026-05-09
draft: false
hideComments: true
tags: ["homelab", "docker", "traefik", "tls", "ansible"]
layout: post
---

Almost a year ago, I set up a tiny homelab on an Intel NUC running Arch Linux. The original goal was fairly modest: run a few self-hosted services for personal use — Plex, Nextcloud, qBittorrent and Bitwarden.

I deliberately avoided Kubernetes. Even though k3s is perfectly capable of running on low-end hardware, I did not want to introduce another layer of complexity into a setup that was supposed to remain small and maintainable. Docker Compose felt more than enough for a single-node environment.

The stack eventually evolved into a collection of Compose services connected through a shared Traefik network. For remote access, I used ZeroTier and exposed services under a `.lan` domain with self-signed TLS.

That worked reasonably well until certificate management became annoying.

## The problem

In Kubernetes, this problem is mostly solved already. You deploy cert-manager, configure an issuer and certificates are rotated automatically.

Docker Compose does not really have an equivalent ecosystem around certificate automation. Traefik can integrate with ACME providers, but that is mainly useful for public domains. For private `.lan` domains and internal-only services, you still need your own CA.

At first, I tried generating certificates with Ansible using the `community.crypto` collection:

The approach itself was fine, and the Ansible [documentation](https://docs.ansible.com/projects/ansible/latest/collections/community/crypto/docsite/guide_selfsigned.html) even includes a guide for self-signed PKI setups.

The problem was lifecycle management.

Generating a CA and leaf certificates is easy. Renewing them automatically later is a different story.

I wanted something closer to an actual internal PKI:

* a dedicated certificate authority
* short-lived certificates
* automatic renewal
* compatibility with Traefik
* no Kubernetes dependency

That eventually led me to [step-ca](https://smallstep.com/docs/step-ca/).

## Why step-ca

`step-ca` is a lightweight certificate authority designed for internal infrastructure. It supports:

* ACME
* automated certificate renewal
* internal PKI management
* proper certificate lifecycles
* lightweight deployment

Most importantly, it works perfectly fine outside Kubernetes.

Instead of reinventing certificate rotation with Ansible, I could simply let step-ca behave like a real internal CA and issue certificates dynamically.

I also wanted to preserve my existing CA certificate rather than rebuilding trust from scratch across all devices in my network.

## Architecture

The final setup ended up looking roughly like this:

![Homelab architecture with Traefik and step-ca](/images/step-ca.png)

Traefik acts as the ingress proxy for all internal services. `step-ca` issues and renews TLS certificates for Traefik automatically.

All services remain attached to the same Docker network.

## step-ca deployment

I deployed `step-ca` as another Docker Compose service managed through Ansible. But before writing actual Ansible tasks, I needed to initialize `step-ca` with the existing PKI.

The official [smallstep/step-ca](https://hub.docker.com/r/smallstep/step-ca) Docker image supports importing an existing root CA during the initial bootstrap process. To do this, the following files need to be mounted into the container:

```text
/run/secrets/root_ca.crt
/run/secrets/root_ca_key
/run/secrets/root_ca_key_password
```

If these files are present, `step-ca` imports the existing CA automatically during its first initialization. One important detail is that these files are only used once during the first init.

The initial bootstrap process itself is fairly straightforward:

```bash
docker run -it -v step:/home/step \
    -p 9000:9000 \
    -e "DOCKER_STEPCA_INIT_NAME=Smallstep" \
    -e "DOCKER_STEPCA_INIT_DNS_NAMES=localhost,$(hostname -f)" \
    -e "DOCKER_STEPCA_INIT_REMOTE_MANAGEMENT=true" \
    smallstep/step-ca
```

During initialization, `step-ca` outputs several important values:

* the CA fingerprint (SHA256)
* the remote management super admin username
* the remote management password

The CA fingerprint is especially important because it is later used by clients during `step ca bootstrap` trust establishment.

Once you've noted the values, you can stop this container and proceed with adding the Ansible configuration.

The Ansible manifest used to configure the service is available [on my GitHub](https://github.com/laslopaul/nuc-arch-mediacenter/blob/master/roles/docker/tasks/step-ca.yml).

## Integrating with Traefik

Traefik was already acting as the reverse proxy for the homelab, so the next step was configuring it to request certificates from step-ca.

The Traefik configuration is available [on my GitHub](https://github.com/laslopaul/nuc-arch-mediacenter/blob/master/roles/docker/tasks/traefik.yml).

Instead of using Let's Encrypt, Traefik points to the internal ACME endpoint exposed by step-ca.

This gives me:

* valid TLS inside the homelab
* automatic renewal
* no browser warnings after trusting the CA
* fully internal infrastructure

Another thing I appreciated about `step-ca` was how easy it made distributing the internal CA certificate to client devices. Instead of manually importing certificates into every system trust store, the step CLI can bootstrap trust automatically:

```bash
step ca bootstrap \
  --ca-url https://step-ca.lan:9000 \
  --fingerprint <ca_fingerprint> \
  --install
```

On Linux systems, this is often enough to make browsers, curl and other tooling trust the internal PKI immediately. For a homelab setup with multiple laptops and devices connected through ZeroTier, this turned out to be significantly cleaner than manually managing self-signed certificates everywhere.

Since everything operates over ZeroTier, services remain accessible remotely without exposing anything publicly.

## Certificate renewal

The part I originally struggled with when using raw Ansible-generated certificates was renewal.

With `step-ca`, this becomes significantly cleaner.

I ended up adding a small [systemd service](https://github.com/laslopaul/nuc-arch-mediacenter/blob/master/roles/docker/templates/traefik-cert-renewer.service.j2) responsible for renewing Traefik certificates periodically.

This keeps certificates short-lived without requiring manual intervention.

## Final thoughts

In retrospect, `step-ca` solved exactly the problem I had:

* internal TLS
* automated renewals
* no Kubernetes
* minimal operational overhead

For small homelab environments running Docker Compose, it fills a gap between “completely manual self-signed certificates” and “full Kubernetes cert-manager ecosystem”.

I still think Docker Compose is the right tradeoff for tiny single-node setups. Kubernetes brings excellent tooling around PKI and ingress management, but for a single Intel NUC running a handful of services, the operational cost rarely feels justified.

`step-ca` gave me most of the certificate management benefits without requiring the rest of the Kubernetes stack.
