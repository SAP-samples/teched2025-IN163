# IN163 - Implementing Exactly Once In Order delivery in SAP Integration Suite
This default template for SAP Samples repositories includes files for README, LICENSE, and REUSE.toml. All repositories on github.com/SAP-samples will be created based on this template.

## Description

This repository contains the material for the SAP TechEd 2025 session called IN163 - Implementing Exactly Once In Order delivery in SAP Integration Suite.

## Overview

This session allows you to implement integration scenarios supporting Exactly Once In Order delivery to ensure data consistency when exchanging messages via SAP Integration Suite, that is messages are delivered without duplication and in their original sequence. In the session we will focus on two options:
- Using exclusive queues in Cloud Integration
- Via partitioned queues of SAP Integration Suite, advanced event mesh

For an introduction into Exactly Once In Order support in Cloud Integration, check out this [blog post](https://community.sap.com/t5/integration-blog-posts/ensuring-exactly-once-in-order-quality-of-service-in-cloud-integration/ba-p/14180026).

For more details about the actual implementaiton options of Exactly Once In Order in Cloud Integration, refer to the [design guidelines](https://help.sap.com/docs/integration-suite/sap-integration-suite/quality-of-service-exactly-once-in-order).

## System Access

For running through the exercises, we provide the following SAP Integration Suite tenants. The instructors will let you know which of the tenants are actually used for the exercises.

Tenants in EU:
<!-- - Option 1: [**Workshop tenant eu-02a**](https://workshop-eu-02a.integrationsuite-cpi033.cfapps.eu10-005.hana.ondemand.com/shell/home). -->
- Option 1: [**Workshop tenant eu-02b**](https://workshop-eu-02b.integrationsuite-cpi035.cfapps.eu20-001.hana.ondemand.com/shell/home).

Tenants in US:
<!-- - Option 3: [**Workshop tenant us-01b**](https://workshop-us-01b.integrationsuite.cfapps.us20.hana.ondemand.com/shell/home). -->
- Option 2: [**Workshop tenant us-01c**](https://workshop-us-01c.integrationsuite.cfapps.us30.hana.ondemand.com/shell/home).

Access:
- Use the user **userXX** with **XX** from 01 to 45.

**Note**: You can copy and paste the user name from below. Please replace **XX** with the ID assigned to you. The ID and the password will be provided by the instructors during the session.

```yaml
userXX
```

For running through Exercise 2, you need to access an **Advanced Event Mesh broker** in addition to the SAP Integration Suite tenant. We will use the following broker:
- [**AEM broker**](https://eu10.console.pubsub.em.services.cloud.sap/login?tenant-id=8b4a1697-2b58-4571-a986-1377cc070073).
- Use the user **IN163-XXX@education.cloud.sap** with **XXX** from 001 to 045.

**Note**: You can copy and paste the user name from below. Please replace **XXX** with the ID assigned to you. The ID and the password will be provided by the instructors during the session.

```yaml
IN163-XXX@education.cloud.sap
```

## Exercises

As a prerequisite for the exercises, you first need to run through the following preparation steps.
- [Exercise 0 - Prepare the Exercises](exercises/ex0/)

The exercises are divided into the following two parts:
- [Exercise 1 - EOIO via Exclusive Queues on Cloud Integration](exercises/ex1/)
- [Exercise 2 - EOIO via Partitioned Queues using AEM](exercises/ex2/)


## Contributing
Please read the [CONTRIBUTING.md](./CONTRIBUTING.md) to understand the contribution guidelines.

## Code of Conduct
Please read the [SAP Open Source Code of Conduct](https://github.com/SAP-samples/.github/blob/main/CODE_OF_CONDUCT.md).

## How to obtain support

Support for the content in this repository is available during the actual time of the online session for which this content has been designed. Otherwise, you may request support via the [Issues](../../issues) tab.

## License
Copyright (c) 2025 SAP SE or an SAP affiliate company. All rights reserved. This project is licensed under the Apache Software License, version 2.0 except as noted otherwise in the [LICENSE](LICENSES/Apache-2.0.txt) file.
