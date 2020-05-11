
# yamltab

yamltab converts Kerberos keytabs to YAML/JSON and the other way around.

## Keytab input, YAML/JSON output

Given a keytab file, yamltab will output a YAML representation of its contents, suitable for human edition

```console
$ yamltab example.keytab
version: 2
entries:
 - spn: HTTP/application.example.com@EXAMPLE.COM
  principal:
    name_type: KRB5_NT_PRINCIPAL
    components:
    - HTTP
    - application.example.com
    realm: EXAMPLE.COM
  kvno: 42
  date: '2020-04-28T14:04:12+0000'
  enctype: AES256_CTS_HMAC_SHA1_96
  key: 0123456789abcdeffedcba9876543210abcdeffedcba9876543210abcdef1234
```

JSON output is also supported:
```console
$ yamltab -f json example.keytab | jq '.entries[0].principal.realm'
"EXAMPLE.COM"
```

## Data layouts
yamltab offers several "data layouts" for its YAML/JSON output:

 - **raw** simply reflects the raw, complete binary structure: it reflects records and holes, tails (i.e. data in a record beyond its entry length) and exposes raw values (name types, timestamps, encryption types, etc.), without any kind of interpretation.
 - **full** is the same structure as *raw* with additional properties for the sake of readability.
 - **simple**, shown above, is the default format; it aims at exposing everything that matters in the keytab  in a user-friendly way.

```console
$ yamltab -l raw -f json example.keytab | jq '.records[0].entry.timestamp'
1588082652
$ yamltab -l full -f json example.keytab | jq '.records[0].entry.date'
"2020-04-28T14:04:12+0000"
$ yamltab -l simple -f json example.keytab | jq '.entries[0].date'
"2020-04-28T14:04:12+0000"
```

## YAML/JSON input, keytab output
yamltab also offers the ability to convert YAML/JSON input into a binary keytab:
```console
$ yamltab example.keytab.yaml | file -
/dev/stdin: Kerberos Keytab file, realm=EXAMPLE.COM, principal=HTTP/application.example.com, type=1, date=Tue Apr 28 14:04:12 2020, kvno=42
```
It supports all three data layouts: *raw* is useful to control every detail of the keytab; *full* is processed the same way. Internally, *simple* input is converted to the *full* data layout; the resulting structure can be inspected using `--simple-to-full`.

## Other options
`--names-types` and `--enc-types` list known name and encryption types, respectively. They can be combined with `--format`:
```console
$ ./yamltab -f json --enc-types | jq '.AES256_CTS_HMAC_SHA1_96'
18
```
`--v1-endianness` exists as a vague attempt to support keytab v1 format. However, keytab v2 has been around since 1992 and real-life keytab v1 files are nowhere to be found. Consequently, keytab v1 support is completely untested and very likely to result in a stacktrace or absurd output.

## Keytab edition
Since yamltab can convert a keytab to YAML and this YAML representation back into a keytab, then it should be easy to assemble a wrapper that allows interactive edition of a keytab file as YAML by invoking your favourite editor. This is what the `keytab-editor` script does.

## Subtleties
The simple format is liable to expose two overlapping properties: `spn` and `principal`. (the former is convenient, the latter is comprehensive). When processing its YAML input, yamltab takes `principal` into account and ignores `spn`... unless `principal` happens to be missing, in which case yamltab attempts to convert `spn` into a KRB5_NT_PRINCIPAL principal.

`name_type_raw`, `enc_type_raw` and `timestamp` override `name_type`, `enc_type` and `date` respectively.

Date handling is NOT clever. Stick to the `YYYY-mm-DDTHHMMSS+ZZZZ` format. That said, it is possible to  write `date: now` in simple mode.

In *simple* mode,`kvno_in_tail: True` can be used to force storage of the key version number as a 32-bit unsigned integer in the tail (i.e. after the entry). `extra_tail` can be used to inject arbitrary data in the tail.
In *full* mode, the following keys strive to reflect kvno:

 - `kvno`: 8-bit key version number found in the entry, always present;
 - `tail_kvno`: 32-bit key version number found right after the entry, present only if actually found;
 - `actual_kvno`: kvno value that matters, always present.

Limitations: yamltab spares you the pain of hex-editing keytab files; however, it offers no encryption-related features. Keys are always handled as hexadecimal blobs, and password hashing is simply not supported.
