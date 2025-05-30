# Cryptography

EscoDB is a document store designed to keep its contents private. It achieves
this via two key features: it lets the user choose where its contents are
physically stored, and it encrypts everything input by the user, including
document IDs (referred to as _paths_ in the API).

It also aims to protect stored data from certain forms of tampering. While
encryption prevents an attacker from being able to read the contents of your
files, tamper resistance means they cannot deface data in certain ways, even if
they are unable to decrypt it. They key thing that EscoDB prevents is the
ability to move encrypted records inside a file so that a document becomes
readable via a different ID than the one it was written to. This is achieved by
appropriate use of _authenticated encryption with associated data_ (AEAD).

EscoDB cannot protect you from arbitrary changes to its data files, including
partial or total deletion of their content. You should still use appropriate
measures to prevent unauthorised access to data in transit and at rest, and you
should back up your data in case it is lost by the primary storage system.

The following pages describe how EscoDB uses cryptography in the data it stores,
including why certain choices have been made.
