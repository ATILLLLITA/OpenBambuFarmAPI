# Disclaimer & responsible-use notice

This repository documents an observed network protocol for the purpose of **interoperability**
— enabling independently-created software to talk to hardware its owner already possesses.

## What this is NOT
- It contains **no** Bambu Lab source code, no binaries, no decompiled output.
- It contains **no** secret cryptographic material: no private keys, no device certificates,
  no TLS session secrets/master secrets, and not the shared farm broker password. Every such
  value is shown only as a `«REDACTED»` placeholder. You cannot impersonate a farm server or a
  printer from anything in this repository.
- "Bambu Lab", "Bambu", model names, and serial numbers are referenced nominatively to
  identify the products being described. No affiliation, sponsorship, or endorsement is implied.

## Intended use
Understanding and reimplementing the farm protocol for your **own** printers and servers
(self-hosting, home automation, archival, research, education). The trademark- and
copyright-relevant facts here are protocol facts (field names, message shapes, ports), which
are not themselves copyrightable.

## Legal note (not legal advice)
Reverse engineering for interoperability is broadly recognized — e.g. EU Software Directive
2009/24/EC Art. 6 (and Art. 8, which voids contract terms that block it), and US case law
(*Sega v. Accolade*, *Sony v. Connectix*, *Google v. Oracle*). Local rules vary, software
EULAs may purport to restrict reverse engineering, and anti-circumvention statutes (e.g. US
DMCA §1201, with its §1201(f) interoperability exception) can be relevant when a protocol is
encrypted. None of this is legal advice. If you intend to publish or build on this, get advice
for your own jurisdiction and use case.

## If you extend this
- Never commit real keys, certs, passwords, or captured session secrets — redact them.
- Don't redistribute vendor binaries or decompiled code.
- Keep claims tied to what you actually observed; mark inference as inference.
