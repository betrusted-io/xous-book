# Backups

Users can make backups of the PDDB, and restore it to the same or new device, by following the [instructions on the wiki](https://github.com/betrusted-io/betrusted-wiki/wiki/Backups).

## Analysis

Offline analysis of a PDDB backup image is now possible thanks to the `backalyzer.py` script, found in the `tools/` directory. This script can take a backup PDDB image, and along with the BIP-39 key, boot password, and any basis name/password combos, decrypt the entire PDDB image, and perform a consistency check of all the Basis structures, dictionaries and keys. It can also be instructed to dump all the data.

This tool could potentially be run in an "air-gapped" environment to securely access PDDB contents from a backup in case a Precursor device is not available or damaged. However, it is not recommended to run this script on a regular, untrusted computer since you will have to share with it all the relevant security credentials to perform an analysis.
