# EUDAMED Audit

This is a community-driven audit of EUDAMED, the European Database on Medical Devices.

## Why Do An Audit?

Our goal is to make it easier for manufacturers to bring medical devices to the market by reducing
bureaucratic overhead and increasing transparency.

The development of EUDAMED is very intransparent. Likewise, the resources allocated to it (funding, key
personnel, etc.) are similarly unknown.

This is bad because, for medical device regulations to actually work, the EU needs functioning infrastructure.

We believe EUDAMED can not be described as "functioning infrastructure".

To add data to this statement, we've created this GitHub repository. It's an open-source, crowd-sourced,
community-driven audit of EUDAMED.

The goals can be summarized as:

* Create transparency around technical shortcomings and hold the EU accountable to fixing them.
* Compare these crowd-sourced audit findings with the *actual EUDAMED audit findings* (it will get audited for
  one year (!)) to hold the EUDAMED auditors accountable.
* Create an actionable list with the goal of improving EU infrastructure.

## Related Links

* [EUDAMED](https://ec.europa.eu/tools/eudamed/#/screen/home)
* [BEUDAMED (our improved version)](https://openregulatory.com/beudamed-better-eudamed/)
* [Our unofficial EUDAMED API docs on GitHub](https://github.com/openregulatory/eudamed-api)

## Contributing

Feel free to open a pull request if you have anything to add. It'd be much appreciated!

## The EUDAMED Audit

### Methodology

* This is a black box audit. In other words, we only analyze the public endpoints of the software because we
  don't have access to the code base.
* We are not professional software auditors - this is based on a rough analysis of the EUDAMED API endpoints.

### Findings

#### 1. Links From Basic UDI DIs To Certificates Are Broken

| Type                       | Severity |
|----------------------------|----------|
| Missing data / duplication | High     |

A Basic UDI-DI, which can be understood as a "high-level" entity of a medical device, can have one or multiple
certificates, e.g. from a technical documentation audit from a notified body.

When looking at Basic UDI-DI data returned from the endpoint, the certificate data is returned as follows:

``` json
{
  "deviceCertificateInfoList": [ ... ]
   // Other keys omitted
}
```

The key `deviceCertificateInfoList` contains an array of certificates relevant to the device (or rather, the
Basic UDI-DI of the device). Each of these entries, has these keys:

``` json
{
  "certificateNumber": "2195-MED-1306401",
  "uuid": "127ad955-98f7-40dc-88e8-7f61764a7051"
  // Other keys omitted
}
```

One could now assume that the relevant certificate can be queried by retrieving it from the certificates
endpoint via its `uuid` key above. However, this is not the case as the endpoint only returns an empty string.

Looking deeper, it turns out the Basic UDI-DI entries are referring to approximately 160k certificates via
unique `uuids`. However, only around 400 certificates exist in total. So there's, at the very least, some
severe data duplication.

Now, one might assume that you could match on the `certificateNumber` instead, which likely must be
unique. However, when looking through all certificates from the certificates endpoint, it turns out that many
certificates are simply missing.

* **Duplication:** Basic UDI-DI entries might return many different `uuid`s which all relate to the same
  certificate
* **Missing data:** Basic UDI-DI entries might return certificates which don't seem to exist in other endpoints
  and can neither be queried by `uuid` nor `certificateNumber`.

#### 2. Certificate Data Is Returned Twice

| Type               | Severity |
|--------------------|----------|
| Data inconsistency | Low      |

When querying Basic UDI-DI entries, certificate data is returned twice in two separate keys:

``` json
{
  "deviceCertificateInfoList": [ ... ]
  "deviceCertificateInfoListForDisplay": [ ... ]
  // Other keys omitted
}
```

Identical certificate data is returned in two keys: `deviceCertificateInfoList` and
`deviceCertificateInfoListForDisplay` for now obvious reason. This goes against software engineering best
practices.

* **Data inconsistency:** Certificate data is returned twice for Basic UDI-DI entries.

#### 3. Authorized Representatives Are Missing

| Type         | Severity |
|--------------|----------|
| Missing data | High     |

When querying the manufacturers endpoint, the API also returns data about the authorized representatives of
the manufacturer:

``` json
{
  "authorisedRepresentatives": [ ... ]
  // Other keys ommitted
}
```

The key `authorizedRepresentatives` contains an array of authorized representatives. Each entry has these
keys:

``` json
{
  "authorisedRepresentativeUuid": "d88f0c00-3829-4be3-a093-a576a6cf0cd1",
  // Other keys ommitted
}
```

One would assume that the endpoint for authorized representatives would return *all* authorized
representatives, including the ones referenced in the above output from the manufacturers endpoint. However,
this is not the case.

It is however possible to query for these separately by querying the authorized representatives endpoint with
the `authorisedRepresentativeUuid`.

* **Missing data:** Manufacturer entries refer to authorized manufacturers which are not returned from the
  authorized manufacturers endpoint and have to be queried separately.

#### 4. Naming Of Nested Data Goes Against Best Practices

| Type          | Severity |
|---------------|----------|
| Inconsistency | Medium   |

Many uuids in nested keys are named arbitrarily, going against programming best practices. Examples are
`authorisedRepresentativeUuid` for authorized representatives and `validatorUuid` for competent
authorities. The keys should be named `uuid` instead.

#### 5. Basic UDI-DI Data Is Kept Separate From Device Data With No Relation

| Type          | Severity |
|---------------|----------|
| Inconsistency | Medium   |

Separate endpoints exist for Basic UDI-DI data and device data, respectively. However, the relation between a
Basic UDI-DI entry and a device entry is not apparent in the data.

Instead, they have to be queried by device:

```
Example endpoint for device data
https://ec.europa.eu/tools/eudamed/api/devices/udiDiData/3a345926-f647-46db-af2b-eb94b4732a2d

Example endpoint for related basic udi-di data (note the same uuid)
https://ec.europa.eu/tools/eudamed/api/devices/basicUdiDiData/3a345926-f647-46db-af2b-eb94b4732a2d
```

Note that, even though the Basic UDI-DI data has to be queried with the device `uuid`, it also has its own
`uuid`. However, the endpoint expects the device `uuid`.

#### 6. Switching From uuid v4 To Self-Generated IDs For Notified Bodies

| Type          | Severity |
|---------------|----------|
| Inconsistency | Medium   |

For almost all entries, the unique key seems to be a UUID v4. However, for some notified bodies, it seems like
EUDAMED has come up with a custom schema for creating unique IDs. Example:

```
URL for TÜV NORD
https://ec.europa.eu/tools/eudamed/#/screen/notified-bodies/63dabd23-5544-4310-9ade-fb225cb3f29f

URL for Intertek (note the different ID schema)
https://ec.europa.eu/tools/eudamed/#/screen/notified-bodies/CD48A569E4688334E05343E4A79E78D8
```

This might be a symptom of a larger problem, e.g. insufficient validation of data on a database-level.

#### 7. Insufficient Performance

| Type        | Severity |
|-------------|----------|
| Performance | Medium   |

When performing a device search, it often takes 10-20 seconds for the API to return a result.

This is not acceptable according to modern standards of web programming.

#### 8. Right-Clicks On Entries Aren't Possible

| Type      | Severity |
|-----------|----------|
| Usability | High     |

Looking at search results, entries can't be right-clicked, e.g. to open them in a new tab. This is due to
entries being `<tr>` (and its nested `<td>`) HTML elements instead of `<a>` elements which would follow best
practices.

It is likely that only a very inexperienced web developer would write HTML in this way.