# `apt-withsha1`: temporary SHA-1 compatibility for APT

Use a separate Sequoia OpenPGP cryptographic policy **only when explicitly running** `apt-withsha1`. Ordinary `apt update` and `apt upgrade` continue using the system's normal policy.

This is intended for third-party APT repositories whose repository metadata is signed with a valid key, but whose signing key contains legacy SHA-1 certifications rejected by newer versions of APT/Sequoia. It is **not** a way to bypass missing or invalid repository signatures.

## Tested environment and scope

Configuration based on Ubuntu 26.10 (Stonking Stingray), APT 3.3.3ubuntu0.1, `sqv` 1.4.0, and the supplied `/usr/share/apt/default-sequoia.config`. The procedure uses APT's `APT_SEQUOIA_CRYPTO_POLICY` environment variable; the wrapper itself has not been executed on your machine.

This exception changes **only** `sha1.second_preimage_resistance` from `2026-02-01` to `2028-02-01`, using the installed APT policy as a starting point. SHA-1 *collision resistance* is **not** enabled. The compatibility cutoff is an expiry date, not a guarantee that all old signing keys will work.

## Installation

These are commands to run **once**, in a terminal. They do not set Bash shell options, modify system APT configuration files, or use a shell alias.

### 1. Create a separate Sequoia policy

```bash
sudo install -d -m 0755 /usr/local/share/apt-withsha1

sudo install -m 0644 \
  /usr/share/apt/default-sequoia.config \
  /usr/local/share/apt-withsha1/sequoia.config

sudo sed -i \
  's/^sha1\.second_preimage_resistance = 2026-02-01\(.*\)$/sha1.second_preimage_resistance = 2028-02-01\1/' \
  /usr/local/share/apt-withsha1/sequoia.config
```

Confirm the intended value is present:

```bash
sudo grep -n '^sha1.second_preimage_resistance = 2028-02-01' \
  /usr/local/share/apt-withsha1/sequoia.config
```

**If the check produces no output, stop:** your installed default policy may use a different value or format. Inspect it before continuing; do not assume the substitution worked.

### 2. Install a standalone command

```bash
sudo tee /usr/local/bin/apt-withsha1 >/dev/null <<'EOF_SCRIPT'
#!/bin/sh
APT_SEQUOIA_CRYPTO_POLICY=/usr/local/share/apt-withsha1/sequoia.config
export APT_SEQUOIA_CRYPTO_POLICY
exec /usr/bin/apt "$@"
EOF_SCRIPT

sudo chmod 0755 /usr/local/bin/apt-withsha1
```

The wrapper deliberately does **not** call `sudo`. It sets the policy path for its own APT process only; the environment variable does not persist in the calling shell.

## Usage

| Command | Behaviour |
|---|---|
| `sudo apt update` | Normal APT signature policy |
| `sudo apt upgrade` | Normal APT signature policy |
| `sudo apt-withsha1 update` | Alternate policy, for this invocation only |
| `sudo apt-withsha1 upgrade` | Alternate policy, for this invocation only |
| `sudo apt-withsha1 full-upgrade` | Alternate policy, for this invocation only |
| `apt-withsha1 --version` | Shows the installed APT version without elevation |

Example:

```bash
sudo apt-withsha1 update
sudo apt-withsha1 upgrade
```

Using the compatibility wrapper for `update` does not require using it for `upgrade`; however, **APT shares cached package indexes** between both commands. Packages obtained from indexes accepted under the relaxed policy may subsequently be considered by normal APT commands. This is not a security isolation boundary.

## Verification

Inspect the policy:

```bash
sudo grep -E 'sha1|signature\.v3' \
  /usr/local/share/apt-withsha1/sequoia.config
```

Verify that the wrapper starts APT:

```bash
apt-withsha1 --version
```

Test the compatibility mode:

```bash
sudo apt-withsha1 update
```

The affected repositories should pass if their **only** remaining problem is the SHA-1 certification cutoff. Other signing failures, including SHA-1 collision-resistance failures, can still occur.

To check that ordinary APT remains strict, run:

```bash
sudo apt update
```

It should still reject *newly checked* metadata that requires the disallowed SHA-1 certification. `Hit:` entries or cached metadata are not proof of a successful fresh signature check.

## Removal

```bash
sudo rm /usr/local/bin/apt-withsha1
sudo rm /usr/local/share/apt-withsha1/sequoia.config
sudo rmdir /usr/local/share/apt-withsha1
```

No system APT policy or repository definition needs to be restored.

## Security considerations

- SHA-1 is deprecated for good reason. This exception accepts some legacy key certifications that the default policy rejects; use it only for repositories you trust.
- Signature verification is **still enabled**. This does not set `trusted=yes`, disable `apt-secure`, or accept unsigned repositories.
- The exception is **invocation-scoped**, but APT's downloaded index cache remains shared.
- The policy is copied at installation time. If a future Ubuntu upgrade changes its default policy, **this copy does not automatically inherit those changes**. Periodically compare it with `/usr/share/apt/default-sequoia.config` and remove the exception once publishers update their signing keys.
- The `2028-02-01` cutoff is a compatibility choice, not a recommended long-term security baseline.

## References

- [APT package release notes — Sequoia policy override](https://fossies.org/linux/apt/debian/NEWS)
- [Sequoia policy configuration syntax](https://docs.rs/sequoia-policy-config/latest/sequoia_policy_config/)
- [Ansible PPA signing-key discussion](https://forum.ansible.com/t/ubuntu-ppa-key-signature/45237)
