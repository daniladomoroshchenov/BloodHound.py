# BloodHound.py — fork for DCs with LDAP signing enforced and no LDAPS

Fork of [dirkjanm/BloodHound.py](https://github.com/dirkjanm/BloodHound.py). Compatible with BloodHound legacy 4.2/4.3.

## The problem

On a domain controller with **LDAP signing enforced** and **no TLS certificate**, stock
`bloodhound-python` 1.9.0 could not authenticate at all: it asked the DC for no SASL security layer,
the DC refused the bind (result code 8, `strongerAuthRequired`), the tool fell back to LDAPS — and
LDAPS is dead on such a DC, because every TLS handshake on 636/3269 is reset. Collection ended with
`LDAPSocketOpenError` and no data, and `--use-ldaps` could not help.

Reproduced on a Windows Server 2025 DC (HackTheBox lab, machine
[Checkpoint](https://app.hackthebox.com/machines/Checkpoint)). The DC itself was healthy: `nxc ldap`
bound with the same credentials and reported `signing:Enforced`, `channel binding:No TLS cert` — the
limitation was in `bloodhound.py`.

IPv6 was not the cause, even though it looked like one: the DC also publishes an AAAA record, and an
old `domain.py` bug overwrote the already verified IPv4 address with it right before the crash. The
only blocker was LDAP signing with no LDAPS.

Example of error:

```text
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: checkpoint.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc01.checkpoint.htb
INFO: Testing resolved hostname connectivity dead:beef::8b2e:c430:c457:a5d2
INFO: Trying LDAP connection to dead:beef::8b2e:c430:c457:a5d2
WARNING: LDAP Authentication is refused because LDAP signing is enabled. Trying to connect over LDAPS instead...
WARNING: Kerberos auth to LDAP failed, trying NTLM
Traceback (most recent call last):
  File "/home/danila/Tools/BloodHound.py/bloodhound.py", line 5, in <module>
    bloodhound.main()
  File "/home/danila/Tools/BloodHound.py/bloodhound/__init__.py", line 347, in main
    bloodhound.run(collect=collect,
  File "/home/danila/Tools/BloodHound.py/bloodhound/__init__.py", line 78, in run
    self.pdc.prefetch_info('objectprops' in collect, 'acl' in collect, cache_computers=do_computer_enum)
  File "/home/danila/Tools/BloodHound.py/bloodhound/ad/domain.py", line 620, in prefetch_info
    self.get_objecttype()
  File "/home/danila/Tools/BloodHound.py/bloodhound/ad/domain.py", line 303, in get_objecttype
    self.ldap_connect()
  File "/home/danila/Tools/BloodHound.py/bloodhound/ad/domain.py", line 114, in ldap_connect
    ldap = self.ad.auth.getLDAPConnection(hostname=self.hostname, ip=ip,
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/danila/Tools/BloodHound.py/bloodhound/ad/authentication.py", line 176, in getLDAPConnection
    return self.getLDAPConnection(hostname, ip, baseDN, 'ldaps')
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/danila/Tools/BloodHound.py/bloodhound/ad/authentication.py", line 164, in getLDAPConnection
    bound = conn.bind()
            ^^^^^^^^^^^
  File "/home/danila/.venv/lib/python3.12/site-packages/ldap3/core/connection.py", line 589, in bind
    self.open(read_server_info=False)
  File "/home/danila/.venv/lib/python3.12/site-packages/ldap3/strategy/sync.py", line 57, in open
    BaseStrategy.open(self, reset_usage, read_server_info)
  File "/home/danila/.venv/lib/python3.12/site-packages/ldap3/strategy/base.py", line 154, in open
    raise LDAPSocketOpenError('invalid server address')
ldap3.core.exceptions.LDAPSocketOpenError: invalid server address
```

## The fix (two changes, one commit)

1. `bloodhound/ad/authentication.py` — the Kerberos LDAP bind now negotiates a SASL security layer
   (integrity + confidentiality, flags `0x3c`) and every following PDU is wrapped in RFC 4121 tokens,
   reusing impacket's `GSSAPI` (same framing as `impacket`'s `LDAPConnection(signing=True)`).
   Works over plain LDAP/389, no TLS involved.
2. `bloodhound/ad/domain.py` — a verified IPv4 DC address is no longer overwritten by an AAAA record;
   IPv6 is used only when no IPv4 address answered.

Verified on the lab stand: exit code 0, full archive (users, groups, computers, domains, gpos, ous,
containers), and no `ldaps`/636 attempt anywhere in the log.

## Install and run

```bash
git clone https://github.com/daniladomoroshchenov/BloodHound.py.git
cd BloodHound.py && git checkout fix/ldap-signing-389
python3 -m venv .venv && source .venv/bin/activate
pip install .

# must print a path inside the clone and True
python3 -c "import bloodhound.ad.authentication as a; print(a.__file__, hasattr(a, 'KerberosSignedSocket'))"

bloodhound-python -c All -d <domain> -u '<user>' -p '<password>' \
  -dc dc01.<domain> -gc dc01.<domain> -ns <dc_ip> --zip
```

* `-ns <dc_ip>` is mandatory: the tool's own resolver reads `/etc/resolv.conf`, so an `/etc/hosts`
  entry alone is not enough.
* Do not add `--use-ldaps` — LDAPS is unavailable on a DC without a certificate.
* Expected result: `INFO: Done in 00M 0xS`, exit code 0, and `<timestamp>_bloodhound.zip` with
  7 JSON files.
* Only `pip install .` from this clone contains the fix; the PyPI package does not.
* Revert: `pip uninstall bloodhound && pip install bloodhound==1.9.0`, or `git checkout master`.

## Known limitation

Only the Kerberos authentication path is patched. The NTLM fallback is untouched, so on a DC where
Kerberos is unavailable and LDAPS is dead, collection will still fail.
