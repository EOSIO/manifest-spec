# EOSIO Manifest Specification

**Version 0.7**

## Why an Application Manifest?
1. Provides user agents metadata about the app for use in application listings (history, favorites, app directories, etc.)
1. Provides app developers with a way to declare which contract actions on a given chain an application on their domain is able to propose to end users
1. Allows user agents to run various transaction pre-flight security assertions comparing the contents of a transaction request with what apps have declared about themselves
1. Provides end users with more security and a better sense of trust--information about the apps they're using is displayed consistently and the transactions themselves are secured with on-chain assertions

## Application Requirements for EOSIO User Agent Compatibility
* The application must register an [application manifest](#application-manifest-specification) on each chain it wishes to transact on.
* Developers must have publicly-accessible `chain-manifests.json` and `app-metadata.json` files.
* The `chain-manifests.json` file must be at the root of the application's declared domain or subdomain.

### How It Works
1. In order for an EOSIO-enabled application to be compatible with EOSIO user agents, developers must submit an [app manifest](#application-manifest-specification) to the EOSIO chains on which it wishes to push transactions.
1. When the application makes a request or proposes a transaction, it must pass along a `declaredDomain`. The user agent will verify the following items. If any fails verification, an error will be displayed to the user or returned to the application. It will verify:
    * that the domain is hosting a `chain-manifests.json` file and that all manifests listed reference the declared domain and app metadata consistently;
    * that the domain is hosting a `app-metadata.json` file and that it contains all required fields;
    * that the domain in the provided manifest matches the domain submitting the transaction (applicable only to web applications);
    * that the application's identifier (bundle ID, package name, etc.) as reported by the OS is whitelisted by the metadata's `appIdentifiers` field (applicable only to native applications);
    * that the contract action(s) in the transaction are whitelisted in the manifest;
    * that the hash of the icon in `app-metadata.json` matches the checksum of the icon file itself;
    * that the hash of `app-metadata.json` matches the hash declared in the provided manifest;
1. When the user accepts the transaction, the user agent will hash the manifest and add an [assertion action](https://github.com/EOSIO/eosio.assert/blob/master/eosio.assert/abi/eosio.assert.abi) to the transaction so that the chain can validate that the manifest provided to the user agent has been registered for the given domain on that chain. If the manifest's hash is not found on the chain, the chain may reject the transaction.

## Specification versioning

Both the application metadata and chain manifest files defined below include fields defining the version of the manifest specification that they follow. Version numbers follow a semantic versioning (semver) inspired scheme consisting of three integers separated by periods `.` and represent the Major, Minor, and Patch levels in the form `M.m.p`. For example `1.2.3`.

Fields are interpretted as follows:

* Patch: Minor changes that only affect the textual description of the specification with no functional changes to schema, generation, or verification. Updates to non-functional parts of the schema like `title` or `description` fields are permitted.
* Minor: Backward-compatible changes to the spec and schema. This would include changes such as adding _optional_ fields or loosening of requirements. A manifest based on spec x.y **MUST** be verifiable against spec x.z where y < z.
* Major: Backward-incompatible changes to the spec and schema. This would include changes such as adding new _required_ fields or tightening of requirements. A manifest based on spec x._ is **NOT** required to be verifiable against spec y._ where x < y.

## Application Metadata Specification
Application metadata must be declared in a publicly available JSON file. The location of this file, along with its checksum hash, will be referenced by the EOSIO chain(s). Having this file referenced on-chain enables EOSIO-compatible user agents to validate the integrity of the metadata and may provide other future benefits. For example, developers could look for these records on the chain and put together public directories of EOSIO-compatible applications.

All of the following fields are required with the exception of `description`, `sslfingerprint`, and `appIdentifiers`.

### Metadata Fields
* `spec_version`: The specification version as described in [Specification Versioning](#specification-versioning).
* `name`: The full name of the application. This will be user-facing in app listings, history, etc.
* `shortname`: A shorter name for the application.
* `scope`: An absolute path relative to the application's root. (`/` or `/app`, but not `../`)
* `apphome`: Tells the browser where your application should start when it is launched. This must fall within the scope.
* `icon`: An HTTPS url or absolute path relative to the application's root followed by a SHA-256 checksum hash. May be displayed in app listings, history, favorites, etc. (`https://landeos.io/icon.png#SHA256HASH` or `/icon.png#SHA256HASH`, but not `../icon.png#SHA256HASH`)
* `appIdentifiers`: Optional. For native applications, an array of whitelisted app identifiers. (e.g., bundle identifiers for iOS apps, package names for Android apps)
* `description`: Optional. A paragraph about your application. May be displayed in app listings, etc.
* `sslfingerprint`: Optional. Your app domain's SSL SHA-256 fingerprint as a hex string. If present, the user agent may check that the SSL fingerprint of the domain submitting the transaction matches that in the provided manifest.
* `chains`: An array containing the `chainId`, `chainName`, and `icon` for any chain for which your application plans to require signing. User agents will use this for presenting a friendly chain name and icon to the user when prompted for signing.

**Note:** While the icon specification is still being developed, for now, these should be PNGs or JPGs at 256px x 256px.

### Example
```json
{
  "spec_version": "0.0.7",
  "name": "Landeos Property Registry",
  "shortname": "Landeos",
  "scope": "/",
  "apphome": "/home",
  "icon": "https://landeos.io/icon.png#SHA256HASH",
  "appIdentifiers": ["io.landeos.registry"],
  "description": "Landeos is a decentralized property registry...",
  "sslfingerprint": "E1 3F 0E 03 2D 05 3C D2 81 FB 57 02 42 51 D0 44 32 08 35 58 8C 73 C4 44 83 4D CA 3E 71 15 16 20",
  "chains": [
    {
      "chainId": "EOSIO1111",
      "chainName": "Property Chain",
      "icon": "/propertyChain.png#SHA256HASH"
    },
    {
      "chainId": "EOSIO2222",
      "chainName": "Other Chain",
      "icon": "/otherChain.png#SHA256HASH"
    }
  ]
}
```

This specification will be extended later to include additional application metadata (e.g., app screenshots and other listing-related data.)

## Application Manifest Specification
Manifests must be registered using the `eosio.assert::add.manifest` action on every chain on which your app will transact. (See: https://github.com/EOSIO/eosio.assert/blob/master/eosio.assert/src/eosio.assert.cpp.) Furthermore, an array of chain manifests must be declared in a publicly available JSON file at the root of the application's `declaredDomain`.

### Manifest Fields
* `account`: The chain account name. This is the account that registered the application manifest.
* `domain`: The uri or bundle identifier.
* `appmeta`: The publicly-accessible location of the application metadata JSON file and its hash. Must be an absolute path.
* `whitelist`: An array containing the `contract` and `action` for each allowed contract action(s) that your app can propose.
    * The `contract` and/or `action` fields within the `whitelist` on the manifest may contain a wildcard. Wildcards are denoted by `""` (empty string). For example, the manifest below whitelists all `eosio.token` actions and another specific contract action.

```javascript
{
  account: string,
  domain: string,
  appmeta: string,
  whitelist: contract_action[]
}
```

**contract_action**
```javascript
{
  contract: string,
  action: string
},
```

### Example
```json
{
  "account": "landeospropr",
  "domain": "https://landeos.io",
  "appmeta": "https://landeos.io/app-metadata.json#SHA256HASH",
  "whitelist": [
    {
      "contract": "eosio.token",
      "action": ""
    },
    {
      "contract": "landeospropr",
      "action": "register"
    }
  ]
}
```

### chain-manifests.json
This file must reside at the root of your application's declared domain. Consists of a top-level object of two properties:

* `spec_version`: The specification version as described in [Specification Versioning](#specification-versioning).
* `manifests`: An array of objects, each of which has two properties:
** `chainId`: The ID of a chain with which the application will transact.
** `manifest`: A manifest as described above in [Application Manifest Specification](#application-manifest-specification)

Here's an example:

```json
{
  "spec_version": "0.0.7",
  "manifests": [
    {
      "chainId": "EOSIO1111",
      "manifest": {
        "account": "landeospropr",
        "domain": "https://landeos.io",
        "appmeta": "https://landeos.io/app-metadata.json#SHA256HASH",
        "whitelist": [
          {
            "contract": "eosio.token",
            "action": ""
          },
          {
            "contract": "landeospropr",
            "action": "register"
          }
        ]
      }
    },
    {
      "chainId": "EOSIO2222",
      "manifest": {
        "account": "landeospropr",
        "domain": "https://landeos.io",
        "appmeta": "https://landeos.io/app-metadata.json#SHA256HASH",
        "whitelist": [
          {
            "contract": "eosio.token",
            "action": ""
          },
          {
            "contract": "landeospropr",
            "action": "register"
          }
        ]
      }
    }
  ]
}
```

## Important

See LICENSE for copyright and license terms.  Block.one makes its contribution on a voluntary basis as a member of the EOSIO community and is not responsible for ensuring the overall performance of the software or any related applications.  We make no representation, warranty, guarantee or undertaking in respect of the software or any related documentation, whether expressed or implied, including but not limited to the warranties or merchantability, fitness for a particular purpose and noninfringement. In no event shall we be liable for any claim, damages or other liability, whether in an action of contract, tort or otherwise, arising from, out of or in connection with the software or documentation or the use or other dealings in the software or documentation.  Any test results or performance figures are indicative and will not reflect performance under all conditions.  Any reference to any third party or third-party product, service or other resource is not an endorsement or recommendation by Block.one.  We are not responsible, and disclaim any and all responsibility and liability, for your use of or reliance on any of these resources. Third-party resources may be updated, changed or terminated at any time, so the information here may be out of date or inaccurate.
