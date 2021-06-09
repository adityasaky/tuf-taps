* POUF: 2
* Title: Reference Implementation Using DSSE
* Version: 1
* Last-Modified: 09-Jun-2021
* Author: Aditya Sirish A Yelgundhalli
* Status: Draft
* TUF Version Implemented:
* Implementation Version(s) Covered:
* Content-Type: text/markdown
* Created: 09-Jun-2021

# Abstract

This POUF describes a proposal to switch the TUF reference implementation maintained by NYU to using Dead Simple Signing Envelope (DSSE).

# Protocol

Refer to POUF-1.

# Operations

Refer to POUF-1.

# Usage

Refer to POUF-1.

# Formats

## General Principals

All signed metadata objects have the format defined in DSSE v1:

       {
         "payload": "<Base64(SERIALIZED_BODY)>",
         "payloadType": "<PAYLOAD_TYPE>",
         "signatures": [{
           "keyid": "<KEYID>",
           "sig": "<Base64(SIGNATURE)>"
         }]
       }

   where:

          * SERIALIZED_BODY is a dictionary whose "_type" field describes the role type.

          * PAYLOAD_TYPE is a fixed as "application/vnd.tuf+json identifying it as TUF metadata.

          * KEYID is the identifier of the key signing the ROLE dictionary.

          * SIGNATURE is a hex-encoded signature of the canonical JSON form of ROLE.

For key formats, refer to POUF-1.


## File Formats

Refer to POUF-1.

# Security Audit

The parts of this profile borrowed from POUF-1 were included in TUF security audits available at https://theupdateframework.github.io/audits.html. The new signature wrapper has not yet been audited.

# Version History

N/A